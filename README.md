# BI Labor - Hadoop

## Reminder

### Apache NiFi - [NiFi](https://nifi.apache.org)

Apache NiFi is software maintained by the Apache Software Foundation that allows you to manage and automate data streams.
The project is very popular, among other reasons, because it can work with many data sources and destinations, as well as provides extensive opportunities for data processing.
Its fields of application are very wide, we will consider it as a kind of ETL tool that helps to load and pre-process data from different sources.

#### Important terms

1. *FlowFile:* A FlowFile can essentially be thought of as a packet that travels along each stream in the system. Each FlowFile consists of two elements, the attributes that contain the metadata and the content of the data that belongs to the FlowFile.
2. *FlowFile Processor:* The essential work is done by the Processors. Their task may be to transform, route, or load the data into an external system. Processors also have access to the attributes and contents of FlowFiles. In the graphs of the NiFi stream, these are the nodes.
3. *Connection:* Each Processor must be connected in some way, the Connections help. In order to be able to connect Processors operating at different speeds, the connections between them also act as a kind of queue, the parameters of which can be configured.
4. *Flow Controller:* Acts as a scheduler that manages the threads and resources reserved for each Processor.
5. *Process Group:* A processing unit that can contain Processors and Connections. You can receive or send data through the Input and Output ports. It is typically used to integrate processing elements moving at different levels of abstraction.

### Superset - [Superset](https://superset.incubator.apache.org/)

Superset is a web-based dashboard builder that can use SQL data sources. With its help, we can create spectacular statements in a graphical way, even without significant SQL knowledge. Its development began at Airbnb. Main entities in the system:

* Dashboard - Slice interface, grouping of slices
* Chart - Basic unit, chart
* Table - Database table
* Database

In the application, charts are defined after the source database and its tables have been added. By assigning charts to dashboards, we can create different surfaces. The tool is great for creating dashboards that provide data to non-IT customers. The created interfaces can be easily embedded in individual web pages.

### Zeppelin - [Zeppelin](https://zeppelin.apache.org/)

Apache Zeppelin is a web-based notebook tool. Thanks to its easy-to-expand architecture, there are many interpreters available to run SQL queries, Python or even Spark code snippets. The tool is excellent for data exploration tasks and experimentation.

## Common part

### Task 0. - initializing the environment

During the lab, all the services will be run as Docker containers using Docker Compose, so they should be available in our environment.

Download and install [Docker Desktop](https://www.docker.com/products/docker-desktop).

Docker is a container-based, small overhead virtualization technology. 
With its help, we can launch Docker containers from Docker Images, which contain a service or software. 
With a few basic commands, you can manage these from the terminal.

* ```docker ps``` - list of running containers
* ```docker exec -it <container name> bash``` - opens a terminal in the given container.
* [More useful commands.](Https://devhints.io/docker)

We will use the following containers:

* MySQL
* NiFi
* Superset
* Zeppelin

During the distance learning, use the docker-compose.yml file. Download this file and in its folder, you can start the Docker container with the following commands:
```sh
docker-compose -p bilabor up -d

docker exec -it bilabor-superset-1 superset-init
```

> Attention! In older Docker Desktop versions the container name separator character is not - but: _

At first startup, the first command downloads the required images and then initializes and starts the four services based on the `docker-compose.yml` file. [More details](https://docs.docker.com/compose/compose-file/compose-file-v3/)

You can see that the ports of the Superset, Zeppelin and NiFi default `8088` and `8080` ports are connected to the `16000`, `16001` and `16002` ports on our own machine.
(To avoid port collisions with possible local instances and previous Docker history.
In the event of a collision, the lines `X:Y` in the `docker-compose.yml` file have to be rewritten to the desired port `X` and the container should be restarted.
In this case, the new port numbers should be used in the browser when opening the UI of the Superset, Zeppelin and NiFi.)

The root user password is set on MySQL and the database also. 

The second command will initialized the Superset, including the admin username and password, which we will use to login to the UI, later.

**Attention! In each task, the inserted screenshots in the report should show the date and time (eg. on the taskbar) and the Name-Neptune code pair (eg. in Notepad).
**

### Task 1. - Data loading with Apache NiFi

[NiFi UI](http://localhost:16002/nifi/)

In the `data` folder of the repository you can find three datasets extracted from the movie database at [http://movielens.org](http://movielens.org) and the associated ratings.
We will work with these datasets during the lab.
Check the datasets, their structure, IDs, and separator characters. [MovieLens Summary](https://github.com/bi-labor/Hadoop/blob/master/data/README)

We need to download the data files to the NiFi container by running the following commands line by line:

```sh
docker exec -it bilabor-nifi-1 bash

cd ..

mkdir movies
mkdir ratings
mkdir users

cd movies

wget https://raw.githubusercontent.com/bi-labor/Hadoop/master/data/movies.dat

cd ..
cd ratings

wget https://raw.githubusercontent.com/bi-labor/Hadoop/master/data/ratings.dat

cd ..
cd users

wget https://raw.githubusercontent.com/bi-labor/Hadoop/master/data/users.dat

```

Downloading MySQL driver

```sh
cd ..

mkdir mysql
cd mysql

wget https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.48/mysql-connector-java-5.1.48.jar

exit
```

#### Task 1.1 - Loading Movies dataset

The first dataset contains data from some popular movies.
Using Apache NiFi, load the contents of the file into MySQL, in the `movies` table. Separator character is ::

The first step is to create the appropriate database table:

```sh
docker exec -it bilabor-db-1 bash

mysql -uroot -proot

use hadooplabor;
```

```sql
CREATE TABLE movies (
    id int,
    title varchar(255),
    genres varchar(255),
    PRIMARY KEY(id)
);
```

For data loading, we will create a simple flow in NiFi. The `GetFile` processor is responsible for reading the data, while the `PutSQL` processor is responsible for writing the data. These can be added to the Nifi interface by dragging the 'Processors' icon on the top toolbar to the canvas. The pop-up window lists all available processor types, from which we need to find the right one.
Configure these so that `GetFile` observes the `/opt/nifi/movies` folder. To access the settings, double-click on a processor or right-click -> configure.

![Flow](screens/nifi/getfile-settings.png)

After reading the file, split it into lines with the `SplitText` processor.

![Flow](screens/nifi/splittext-properties.png)

Each line must be converted to SQL INSERT statements, using the `ReplaceText` processor, but first the values in the line must be converted to a FlowFile attribute with the `ExtractText` processor. Regexes used for ExtractText (new items can be added with the + button in the upper right corner of the Properties tab):

* genres: `[0-9]+::.*::(.*)`
* movieId: `([0-9]+)::.*::.*`
* title: `[0-9]+::(.*)::.*`

![Flow](screens/nifi/extracttext-properties.png)

Set the Replacement Startegy value of the `ReplaceText` processor to Always Replace and the evaluation mode to Entire Text.

![Flow](screens/nifi/replacetext-properties.png)

The replacement value:

```sql
INSERT INTO hadooplabor.movies (id,title,genres) VALUES (${'movieId'},'${'title'}','${'genres'}');
```

*Expressions enclosed in ${} are elements of the NiFi expression language, with this form we can replace FlowFile attributes.*

The completed INSERT statements can be run with the PutSQL processor and our data records will be saved in the database. The PutSQL processor needs a NiFi server for the DB connection with its settings:

* **connection URL**: jdbc:mysql://db:3306/hadooplabor
* **Driver Class Name**: com.mysql.jdbc.Driver
* **Driver location**: /opt/nifi/mysql
* **User**: root
* **Password**: root
* **Support Fragmented Transactions**: false

To configure, add a new database service in the processor Properties tab.

![Flow](screens/nifi/db-setup-1.png)

Then click on the right arrow next to the new service.
![Flow](screens/nifi/db-setup-2.png)

For the 1 controller service listed, select the settings with the gear icon and enter the required data.
![Flow](screens/nifi/db-setup-3.png)

Finally, we activate the database connection with the lightning icon.
![Flow](screens/nifi/db-setup-4.png)

After that, all we have to do is connect our processors, creating the Connections. We can do this with the mouse. Hovering the mouse over a processor will display an arrow icon, it must be dragged to the target processor. In the pop-up window, select which output of the processor you want to connect. It is very important that unused outputs be marked autoterminate, in the first tab of the processor settings view, or the processor will not start. If you have unconnected and non autoterminated output, this is also indicated by a yellow triangle on the processor.

Autoterminate outputs: ![Flow](screens/nifi/splittext-autoterminate.png)

The name of the selected output can be read on the small box that appears on the connection, this is also shown in the figure below, based on this the flow must be set. Completed total flow:  ![Flow](screens/nifi/flow.png)

When we're done, we can start the processors. We can do this one by one or all at once. You can start and stop the processors in the right-click menu on the processor, or the selected processors can be started at the same time on the left side of the canvas.

**Note:** There will be errors with the SQL insert because we did not escape the apostrophe and quotation mark characters. This is not a problem now. Can be easily solved with ReplaceText.

Through the flow, we can track what happens to our file. Each processor prints to the interface how many records were received and passed on. This is most spectacular in splittext where 1 FlowFile goes in and 3884 comes out. If we look at the movies.dat file, it had just that many lines, so we can be sure that SplitText worked well.

*Check:* Place 1 screenshot in the report of the created flow  and 1 screenshot in the report of the MySQL window about the records appeared (3426 lines).

#### Task 1.2 - Loading Ratings dataset

The necessary SQL data table:

```sql
CREATE TABLE ratings (
    userId int,
    movieId int,
    rating int,
    timestamp varchar(255),
    PRIMARY KEY (userId, movieId)
);
```

In order to make our NiFi Flow configuration more transparent, let’s create a new Process Group where we copy the existing Processors.
In addition, create another Process Group for the current task.

Here, too, we will follow a similar solution as the previous ones.

> Attention! In the Ratings data file, the separator character is not :: but !

Assemble this Flow as well, then check the result obtained!

*Check:* Place 1 screenshot in the report of the created flow  and 1 screenshot in the report of the MySQL window about the records appeared.

### Task 2. - Zeppelin data exploration

[Zeppelin UI](http://localhost:16001/#/)

#### Zeppelin setup

In order for Zeppelin to access our database, the MySQL driver and database settings must be included. With these we create a new JDBC interpreter, but first we also need to add an unsecure maven repository to Zeppelin.

[Settings](https://zeppelin.apache.org/docs/0.8.0/interpreter/jdbc.html)

Select the Interpreter option by clicking on anonymous at the top right.

![Interpreters](screens/zeppelin/interpreters.png)

We first add the maven repo, select repositories at the top right, then the + button next to the list that appears. Fill in the form as shown in the picture and save it. The mvn repository url used:

```sh
http://insecure.repo1.maven.org/maven2/
```

![Interpreters](screens/zeppelin/maven.png)

Then press the create button at the top right and add a new jdbc interpreter with the data below.

* **default.driver**: com.mysql.jdbc.Driver
* **default.password**: root
* **default.user**: root
* **default.url**: jdbc:mysql://db:3306/hadooplabor
* **artifact**: mysql:mysql-connector-java:jar:5.1.45

![Interpreters](screens/zeppelin/new1.png)
![Interpreters](screens/zeppelin/new2.png)

Save with the save button at the bottom.

#### Some simple queries

Let’s create a new notebook with our newly added interpreter and try some simple queries.

List of action movies:

```sql
SELECT * FROM movies WHERE genres LIKE '%Action%';
```

Distribution of ratings:

```sql
SELECT rating, count(*) FROM ratings GROUP BY rating;
```

Number of movies by genre:

```sql
SELECT count(*) as count, genres from movies group by genres order by count desc
```

*Check:* Place 1-1 screenshot of the results of the queries in the report.

### Task 3. - Superset dashboards

[Superset UI](http://localhost:16000/login/)

#### Task 3.1 - Connecting to database

Entering Superset in the Sources / Databases interface, the + button is used to add a new data source.

* Database: hadooplabor
* SQLAlchemy URI: mysql://root:root@db:3306/hadooplabor
* Expose in SQL Lab: true

![Interpreters](screens/superset/database.png)

Save this at the bottom with the save button. If Superset does not fetch the tables by itself, our 3 tables must be added separately in the Sources / tables menu in a similar way.

You can add a new chart by clicking on the Charts menu, where the wizard will guide you through the steps. Importantly, we can only work from 1 table in this wizard interface. If you want to join tables, you have to manually write the SQL query in SQL Lab, and then you can enter its result into the chart editor like a view.

Make the same statements as in Zeppelin!

*Check:* Place 1-1 screenshot of the results of the queries in the report.

## Individual tasks

### Task 1. - Loading Users dataset with Apache NiFi

Also load the `users` dataset to the MySQL database!
Filter users under age of 18 during loading.
A description of the data structure can be found in the repository's `data/README` file. **ATTENTION! The separator character: ,**

Tips:

* The [RouteText](https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi/nifi-standard-nar/1.5.0/org.apache.nifi.processors.standard.RouteText/index.html) processor can be useful for filtering under age of 18.
* It is suggested to solve the filtering with regular expression or [NiFi Expression Language](https://nifi.apache.org/docs/nifi-docs/html/expression-language-guide.html) by adding a new property.

*Check:* Place 1 screenshot in the report of the created flow  and 1 screenshot in the report of the MySQL window about the records appeared.

### Task 2. - Zeppelin analysis

Create arbitrary analyzes on the data set with Zeppelin. Make at least two analyzes. Use different chart types. Some tips:

* Write a query that lists the titles, IDs, and number of ratings for the 10 most rated movies! Visualize the results!

* Write a query that lists the titles, IDs, and number of ratings received for the 10 most 1-rated movies! Visualize the results!

* Write a query that lists the titles, IDs, and number of ratings received by programmers for their 3 favorite 5-rated movies! (Which received the most 5-rates) Visualize the results!

*Check:* Place a screenshot of eaxh analysis and explain in 1-1 sentence what we can see on the picture.

### Task 3. - Superset analysis

Create arbitrary analyzes on the dataset using Superset. Make at least two analyzes. Use different chart types. The previous examples can be used, and it’s even good to do the same two analyzes with both tools. Visualize the results!

*Check:* Place a screenshot of eaxh analysis and explain in 1-1 sentence what we can see on the picture.

## Evaluation

Finishing the common part is the minimum for mark 2, the solution of the 3 individual tasks means each +1 grade, ie:
* Common part + 1 solved individual task = 3
* Common part + 2 solved individual tasks = 4
* Common part + all solved individual tasks = 5

