# BI Labor - Hadoop

## Emlékeztető

### Apache NiFi - [NiFi](https://nifi.apache.org)

Az Apache NiFi egy az Apache Software Foundation által karbantartott szoftver, mely segítségével adatfolyamokat menedzselhetünk és automatizálhatunk.
A projekt igen népszerű, többek között azon oknál fogva, hogy számos adatforrással és célponttal tud dolgozni, valamint kiterjedt lehetőségeket biztosít az adatok feldolgozására is.
Felhasználási területei igen széleskörűek, mi egyfajta ETL eszközként fogunk rá tekinteni, amely segít az adatok különböző forrásokból történő betöltésében, előfeldolgozásában.

#### Fontos fogalmak

1. *FlowFile:* Egy FlowFile lényegében egy csomagként fogható fel, amely a rendszerben halad az egyes adatfolyamok mentén. Minden FlowFile két elemből áll össze, a metaadatokat tartalmazó attribútumokból, és a FlowFilehoz tartozó adat tartalmából.
2. *FlowFile Processor:* A lényegi munkát a Processorok végzik el. Feladatuk lehet az adat transzformálása, routeolása, vagy betöltése valamilyen külső rendszerbe. A Processorok hozzáférnek a FlowFileok attribútumaihoz, és tartalmához is.
3. *Connection:* Az egyes Processorokat valamilyen módon össze kell kötni, ebben segítenek a Connectionök. Annak érdekében, hogy a különböző sebességgel működő Processorok összeköthetők legyenek, a köztük lévő kapcsolatok egyfajta várakozási sorként is működnek, melyek paraméterei konfigurálhatók. 
4. *Flow Controller:* Egyfajta ütemezőként működik, amely az egyes Processorok számára fenntartott szálakat és erőforrásokat kezeli.
5. *Process Group:* Feldolgozási egység, amely tartalmazhat Processorokat és Connectionöket. Fogadhat, illetve küldhet adatot az Input és Output portjain keresztül. Tipikusan a különböző absztrakciós szinten mozgó feldolgozási elemek egységbe foglalására használjuk.

### Superset - [Superset](https://superset.incubator.apache.org/)

A Superset egy web alapú dashboard készítő eszköz, ami SQL adatforrásokat tud használni. Segítségével grafikus módon, akár jelentősebb SQL tudás nélkül is látványos kimutatásokat készíthetünk. Fejlesztését az Airbnb nél kezdték. Főbb entitások a rendszerben:

* Dashboard - Sliceokból álló felület, Sliceok (grafikonok) csoportosítása
* Slice - Alap egység, grafikon
* Table - Adatbázis tábla
* Database - Adatbázis

Az alkalmazásban a forrás adatbázis és annak táblái felvételét követően sliceokat definiálunk. A sliceokat dashboardokhoz rendelve különbőző felületeket készíthetünk. Az eszköz remekül használható olyan dashboardok keszítésére melyet nem IT-s ügyfeleknek szolgaltatnak adatokat. Az elkészített felületek akár könnyen egyedi weboldalakba is ágyazhatók.

### Zeppelin - [Zeppelin](https://zeppelin.apache.org/)

Az Apache Zeppelin egy web alapú notebook eszköz. Könnyen bővíthető architektúrájának köszönhetően számos interpreter érhető el hozzá melyekkel SQL lekérdezéseket, Python vagy éppen Spark kódrészleteket futtathatunk. Az eszköz kiválóan alkalmas data exploration feladatokra, kísérletezésre.

## Vezetett rész

### 0. Feladat - környezet előkészítése

A labor során az összes szükséges eszköt Docker konténerként fogjuk futtatni. A következő eszközökre lesz szükség:

* MySQL
* NiFi
* Superset
* Zeppelin

A következő parancsokkal indíthatjuk el őket:

**Otthon:**

```
docker-compose -p bilabor up -d

docker exec -it bilabor_superset_1 superset-init
```

**Egyetemi Labor környezetben:**

Docker beállitásokban felvenni a tanszéki privát Docker Registryt.

```
docker-compose -p bilabor -f docker-compose-aut.yml up -d

docker exec -it bilabor_superset_1 superset-init
```

### 1. Feladat - adatbetöltés Apache NiFivel

[NiFi UI](http://localhost:8080/nifi/)

A repository `data` mappájában megtalálhatunk három adathalmazt, amelyet a [http://movielens.org](http://movielens.org) oldalon található filmadatbázisból, és a hozzá tartozó értékelésekből nyertek ki.
A labor során ezekkel az adathalmazokkal fogunk dolgozni.

Az adatfileokat le kell töltenunk a NiFi konténerébe, ehhez tegyük a következőt:

```
docker exec -it bilabor_nifi_1 bash

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

MySQL driver letöltése

```
cd ..

mkdir mysql
cd mysql

wget http://central.maven.org/maven2/mysql/mysql-connector-java/5.1.45/mysql-connector-java-5.1.45.jar

exit
```


#### 1.1 Feladat - Movies dataset betöltése

Az első betöltendő adathalmaz néhány népszerű film adatait tartalmazza.
Apache NiFi használatával töltsük be a fájl tartalmát MySQL-be, a `movies` táblába. (szeparator karakter: ::)

Első lépésként létre kell hoznunk a megfelelő adatbázistáblákat:

```
docker exec -it bilabor_db_1 bash

mysql -uroot -proot

use hadooplabor;
```

```
CREATE TABLE movies (
    id int,
    title varchar(255),
    genres varchar(255),
    PRIMARY KEY(id)
);
```

Az adatbetöltéshez egy egyszerű workflowt fogunk létrehozni. Az adatok beolvasásárért a `GetFile`, míg az SQL-be írásért a `PutSQL` processzor a felelős.
Konfiguráljuk be ezeket úgy, hogy a `GetFile` a `/opt/nifi/movies` mappát figyelje. A fajl felolvasasat követően bontsuk azt sorokra a `SplitText` processzorral. Az egyes sorokat SQL INSERT statementekké kell alakítanunk, ehhez hasznalhatjuk a `ReplaceText` processzort, de előbb a sorban található értékeket FlowFile attribútummá kell alakítanunk az `ExtractText` processzorral. Az ExtractText-nél használt regexek:

* genres: `[0-9]+::.+::(.+)`
* movieId: `(.*)::.*::.*`
* title: `[0-9]+::(.*)::.*`

A `ReplaceText` processzor Replacement Startegy erteket allisuk Always Replace-re, az evaluation mode ot pedig Entire Text-re.

A replacement value:
```
INSERT INTO hadooplabor.movies (id,title,genres) VALUES (${'movieId'},'${'title'}','${'genres'}');
```

*A ${} koze zart kifejezesek a NiFi expression language elemei, jelen formaval FlowFile attributumokat tudunk behelyettesiteni.*

Az elkészült insert statementeket a PutSQL processzorral lefuttathatjuk es ezzel az adatrekordjaink mentésre kerülnek az adatbázisba. Az PutSQL processzornak szüksége van egy NiFi servicere a DB csatlakozáshoz ennek a beállításai:

* connection URL: jdbc:mysql://db:3306/hadooplabor
* Driver Class Name: com.mysql.jdbc.Driver
* Driver location: /opt/nifi/mysql
* User: root
* Password: root
* Support Fragmented Transactions: false

**Megjegyzés:** Az SQL insertnél lesznek hibák, mert nem escapeltük az aposztróf és idézőjel karaktereket. Ez most nem gond. ReplaceText-el egyszerűen megoldható.

*Ellenőrzés:* A jegyzőkönyvben helyezz el egy képet a létrejött flowról, illetve arról, hogy MySQL-ben megjelentek a rekordok (3426 sornak kell lennie).

#### 1.2 Feladat - Ratings dataset betöltése

A szükséges SQL adattábla:

```
CREATE TABLE ratings (
    userId int,
    movieId int,
    rating int,
    timestamp varchar(255),
    PRIMARY KEY (userId, movieId)
);
```

Annak érdekében, hogy átláthatóbb legyen a NiFi Flow konfigurációnk, hozzunk létre egy új Process Groupot, ahova bemásoljuk az eddigi Processorokat.
Ezen kívül hozzunk létre egy másik Process Groupot is, az aktuális feladat számára.

Itt is hasonló megoldást fogunk követni, mint az előzőekben.

A laborvezető segítségével állítsuk össze ezt a Flowt is, majd ellenőrizzük le a kapott eredményt!

*Ellenőrzés:* A jegyzőkönyvben helyezz el egy képet a létrejött flowról, illetve arról, hogy MySQL-ben megjelentek a rekordok.

### 2. Feladat - Zeppelin data exploration

[Zeppelin UI](http://localhost:8081/#/)

#### Zeppelin setup

Hogy a Zeppelin hozzáférhessen az adatbázisunkhoz, fel kell venni a MySQL drivert és az adatbázis beállításokat. Ezekkel egy új JDBC interpretert készítünk.

[Settings](https://zeppelin.apache.org/docs/0.7.0/interpreter/jdbc.html)

* default.driver: com.mysql.jdbc.Driver
* default.password: root
* default.user: root
* default.url: jdbc:mysql://db:3306/hadooplabor
* artifact: mysql:mysql-connector-java:jar:5.1.45

#### Néhány egyszerű lekérdezés

Akciófilmek listája:
```
SELECT * FROM movies WHERE genres LIKE '%Action%';
```

Értékelések eloszlása:
```
SELECT rating, count(*) FROM ratings GROUP BY rating;
```

Filmek száma műfajonként:
```
SELECT count(*) as count, genres from movies group by genres order by count desc
```

*Ellenőrzés:* A lekérdezések eredményeiről helyezz el egy képernyőképet a jegyzőkönyvben!

### 3. Feladat - Superset dashboardok

[Superset UI](http://localhost:8083/login/)

#### 3.1 Feladat - Kapcsolódás adatbázishoz

Supersetbe belépve a Sources / Databases felületen a + gombbal új adatforrást veszünk fel.

* Database: hadooplabor
* SQLAlchemy URI: mysql://root:root@db:3306/hadooplabor
* Expose in SQL Lab: true

Készítsük el ugyanazokat a kimutatásokat mint Zeppelinben!

## Önálló feladatok

### 1. Feladat - Users dataset betöltése Apache NiFi segítségével

Töltsd be a `users` adatállományt is a MySQL adatbázisba!
A betöltés során szűrd ki a 18 év alatti felhasználókat.
Az adatszerkezet leírása a repository `data/README` fájljában található. A szeparator karekter: ,

Tippek:

* A 18 éven aluliak kiszűréséhez jól jöhet a RouteText processzor

*Ellenőrzés:* A jegyzőkönyvben helyezz el egy képet a létrejött flowról, illetve arról, hogy MySQL-ben megjelentek a rekordok.

### 2. Feladat - Zeppelin, Superset elemzesek

Keszits tetszoleges elemzeseket az adathalmazon Zeppelin es Superset segitsegevel. Osszesen legalabb 4-et, ugyanaz elkeszitve Superetben es Zeppelinben kettonek szamit! Hasznalj kulonbozo grafikon tipusokat. Nehany tipp:

* Írj egy lekérdezést, amely kiírja a 10 legtöbbet értékelt film címét, azonosítóját és a rá érkezett értékelések számát! Vizualizald az eredmenyeket!

* Írj egy lekérdezést, amely kiírja a 10 legtöbb 1-es osztályzattal értékelt film címét, azonosítóját és a rá érkezett 1-es értékelések számát! Vizualizald az eredmenyeket!

* Írj egy lekérdezést, amely kiírja a programozók 3 kedvenc filmjének címét, azonosítóját és a rájuk érkezett 5-ös értékelések számát! (Amelyek a legtöbb 5-ös szavazatot kapták.) Vizualizald az eredményeket!

*Ellenőrzés:* A jegyzőkönyvben helyezz el képernyőképeket az elemzésekről és 1-1 mondatban írd le mit látunk a képen.
