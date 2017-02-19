PostgreSQL & PostGIS Cheatsheet
===============================
This is a collection of information on PostgreSQL and PostGIS for what I tend to use most often.

## TOC
- [Installing Postgres & PostGIS](#installation)  
- [Using Postgres on the command line: PSQL](#psql)
- [Importing Data into Postgres](#importing-data)
- [Exporting Data from Postgres](#exporting-data)
- [Joining Tables](#joining-tables-using-a-shared-key)
- [Upgrading Postgres](#upgrading-postgres)
- [PostGIS common commands](#postgis-1)
- [Common PostGIS spatial queries](#common-spatial-queries)
- [Spatial Indexing](#spatial-indexing)
- [Importing spatial data into PostGIS](#importing-spatial-data-to-postgis)
- [Exporting spatial data from PostGIS](#exporting-spatial-data-from-postgis)
- [Other Methods of Interacting With Postgres/PostGIS](#other-methods-of-interacting-with-postgres/postgis)

## Installation
### Postgres
- to install on Ubuntu do: `apt-get install postgresql`

- to install on Mac OS X first install [homebrew](http://brew.sh/) and then do `brew install postgresql`

- to install on Windows...

Note that for OS X and Ubuntu you may need to run the above commands as a super user / using `sudo`.

#### Set Up
On Ubuntu you typically need to log in as the Postgres user and do some admin things:

- log in as postgres: `sudo -i -u postgres`
- create a new user: `createuser --interactive`
- type the name of the new user (no spaces!), typically the same name as your linux user that isn't root. You can add a new linux user by doing `adduser username`.
- typically you want the user to have super-user privileges, so type `y` when asked.
- create a new database that has the same name as the new user: `createdb username`

For Mac OS X you can skip the above if you install with homebrew.

For Windows....


#### Starting the Postgres Database
On Mac OS X:  

- to start the Postgres server do: `postgres -D /usr/local/var/postgres`

- or do `pg_ctl -D /usr/local/var/postgres -l /usr/local/var/postgres/server.log start` to start and `pg_ctl -D /usr/local/var/postgres -l /usr/local/var/postgres/server.log stop` to stop

- to have Postgres start everytime you boot your Mac do: `ln -sfv /usr/local/opt/postgresql/*.plist ~/Library/LaunchAgents` then to check that it's working after booting do: `ps ax | grep sql`

### PostGIS
- On Ubuntu do `apt-get install postgis`

- On Mac OS X the easiest method is via homebrew: `brew install postgis`  
(note that if you don't have Postgres or GDAL installed already it will automatically install these first).

- to install on Windows...

## psql
psql is the interactive unix command line tool for interacting with Postgres/PostGIS.

### Common Commands
- log-in / connect to a database name by doing `psql -d db_name`

- for doing admin type things such as managing db users, log in as the postgres user: `psql postgres;`

- to create a database: `CREATE DATABASE database-name;`

- to connect to a database: `\c database-name;`

- to delete a database `DROP DATABASE database-name;`

- to connect when starting psql use the `-d` flag like: `psql -d nyc_noise`

- to list all databases: `\l`

- to quit psql: `\q`

- to grant privileges to a user (requires logging in as `postgres` ):

	`GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;`

- to enable the hstore extension ( for key : value pairs, useful when working with OpenStreetMap data) do: `CREATE EXTENSION hstore`

- to view columns of a table: `\d table_name`

- to list all columns in a table (helpful when you have a lot of columns!):  
  `select column_name from information_schema.columns where table_name = 'my_table' order by column_name asc;`

- to rename a column:  
  `alter table noise.hoods rename column noise_sqkm to complaints_sqkm;`

- to change a column's data type:  
  `alter table noise.hoods alter column noise_area type float;`

- to compute values from two columns and assign them to another column: `update noise.hoods set noise_area = noise/(area/1000);`

- to search by wildcard use the `like` (case sensitive) or `ilike` (treats everything as lowercase) command:   
  `SELECT count(*) from violations where inspection_date::text ilike '2014%';`

- to insert data into a table:  

  ```
  INSERT INTO table_name (column1, column2)
  VALUES
  	(value1, value2);
  ```

- to insert data from another table:

  ```
  INSERT INTO table_name (value1, value2)
  SELECT column1, column2
  FROM other_table_name
  ```


- to remove rows using a where clause:  
  `DELETE FROM table_name WHERE some_column = some_value`


- **list all column names from a table in alphabetical order:**  

  ```
  select column_name
  from information_schema.columns
  where table_schema = 'public'
  and table_name = 'bk_pluto'
  order by column_name;
  ```

- **List data from a column as a single row, comma separated:**
  1. `SELECT array_to_string( array( SELECT id FROM table ), ',' )`  
  2.  `SELECT string_agg(id, ',') FROM table`

- **rename an existing table:**  
  `ALTER TABLE table_name RENAME TO table_name_new;`

- **rename an existing column** of a table:  
  `ALTER TABLE table_name RENAME COLUMN column_name TO column_new_name;`

- **Find duplicate rows** in a table based on values from two fields:

	```
	select * from (
	  SELECT id,
	  ROW_NUMBER() OVER(PARTITION BY merchant_Id, url ORDER BY id asc) AS Row
	  FROM Photos
	) dups
	where
	dups.Row > 1
	```
	credit: [MatthewJ on stack-exchange](http://stackoverflow.com/questions/14471179/find-duplicate-rows-with-postgresql)

- **Bulk Queries** are efficient when doing multiple inserts or updates of different values:

	```
	UPDATE election_results o
	SET votes=n.votes, pro=n.pro
	FROM (VALUES (1,11,9),
	             (2,44,28),
	             (3,25,4)
	      ) n(county_id,votes,pro)
	WHERE o.county_id = n.county_id;
	```

	```
	INSERT INTO election_results (county_id,voters,pro)
	VALUES  (1, 11,8),
	        (12,21,10),
	        (78,31,27);    
    ```
    ```
	WITH
	-- write the new values
	n(ip,visits,clicks) AS (
	  VALUES ('192.168.1.1',2,12),
	         ('192.168.1.2',6,18),
	         ('192.168.1.3',3,4)
	),
	-- update existing rows
	upsert AS (
	  UPDATE page_views o
	  SET visits=n.visits, clicks=n.clicks
	  FROM n WHERE o.ip = n.ip
	  RETURNING o.ip
	)
	-- insert missing rows
	INSERT INTO page_views (ip,visits,clicks)
	SELECT n.ip, n.visits, n.clicks FROM n
	WHERE n.ip NOT IN (
	  SELECT ip FROM upsert
	);    
	```
### Importing Data
- import data from a CSV file using the COPY command:  

	```
	COPY noise.locations (name, complaint, descript, boro, lat, lon)
	FROM '/Users/chrislhenrick/tutorials/postgresql/data/noise.csv' WITH CSV HEADER;
	```
- import a CSV file "AS IS" using csvkit's `csvsql` (requires python, pip, csvkit, psycopg2):

	```
	csvsql --db postgresql:///nyc_pluto --insert 2012_DHCR_Bldg.csv
	```

### Exporting Data
- export data as a CSV with Headers using COPY:

	```
	COPY dob_jobs_2014 to '/Users/chrislhenrick/development/nyc_dob_jobs/data/2014/dob_jobs_2014.csv' DELIMITER ',' CSV Header;
	```

- to the current workspace without saving to a file:

	```
	COPY (SELECT foo FROM bar) TO STDOUT CSV HEADER;
	```

- from the command line w/o connecting to postgres:

	```
	psql -d dbname -t -A -F"," -c "select * from table_name" > output.csv
	```


### Joining Tables Using a Shared Key
From CartoDB's tutorial [Join data from two tables using SQL](http://docs.cartodb.com/tutorials/joining_data.html)

- Join two tables that share a key using an `INNER JOIN`(Postgresql's default join type):

	```
	SELECT table_1.the_geom,table_1.iso_code,table_2.population
	FROM table_1, table_2
	WHERE table_1.iso_code = table_2.iso
	```

- To update a table's data based on that of a join:

	```
	UPDATE table_1 as t1
	SET population = (
	  SELECT population
	  FROM table_2
	  WHERE iso = t1.iso_code
	  LIMIT 1
	)
	```

- aggregate data on a join (if table 2 has multiple rows for a unique identifier):

	```
	SELECT
	  table_1.the_geom,
	  table_1.iso_code,
	  SUM(table_2.total) as total
	FROM table_1, table_2
	WHERE table_1.iso_code = table_2.iso
	GROUP BY table_1.iso_code, table_2.iso
	```
- update the value of a column based on the aggregate join:

	```
	UPDATE table_1 as t1
	SET total =  (
	  SELECT SUM(total)
	  FROM table_2
	  WHERE iso = t1.iso_code
	  GROUP BY iso
	)
	```

### Upgrading Postgres
[This Tutorial](http://blog.55minutes.com/2013/09/postgresql-93-brew-upgrade/) was very helpful for upgrading on Mac OS X via homebrew.

**_WARNING:_** **Back up your data before doing this incase you screw up like I did!**

Basically the steps are:  

1. Shut down Postgresql:  
	`launchctl unload ~/Library/LaunchAgents/homebrew.mxcl.postgresql.plist`

2. Create a new Postgresql9.x data directory:  
	`initdb /usr/local/var/postgres9.4 -E utf8`

3. Run the pg_upgrade command:

	```
	pg_upgrade \
	-d /usr/local/var/postgres \
	-D /usr/local/var/postgres9.4 \
	-b /usr/local/Cellar/postgresql/9.3.5_1/bin/ \
	-B /usr/local/Cellar/postgresql/9.4.0/bin/ \
	-v
	```
4. Change kernel settings if necessary:

	```
	sudo sysctl -w kern.sysv.shmall=65536
	sudo sysctl -w kern.sysv.shmmax=16777216
	```  
	- I also ran sudo vi /etc/sysctl.conf and entered the same values:

	```
	kern.sysv.shmall=65536
	kern.sysv.shmmax=16777216
	```
	- re-run the pg_upgrade command in step 3

5. Move the new data directory into place:

	```
	cd /usr/local/var
	mv postgres postgres9.2.4
	mv postgres9.3 postgres
	```
6. Start the new version of PostgreSQL:
	 `launchctl load -w ~/Library/LaunchAgents/homebrew.mxcl.postgresql.plist`  
   - check to make sure it worked:

   ```
   psql postgres -c "select version()"
   psql -l
   ```

7. Cleanup:  
	- `vacuumdb --all --analyze-only`  
	- `analyze_new_cluster.sh`*  
	- `delete_old_cluster.sh`*
	- `brew cleanup postgresql`  
	(* scripts were generated in same the directory where `pg_upgrade` was ran)


## PostGIS
PostGIS is the extension for Postgres that allows for working with geometry data types and doing GIS operations in Postgres.

### Common Commands

- to enable PostGIS in a Postgres database do: `CREATE EXTENSION postgis;`

- to enable PostGIS topology do: `CREATE EXTENSION postgis_topology;`

- to support OSM tags do: `CREATE EXTENSION hstore;`

- create a new table for data from a CSV that has lat and lon columns:

  ```
  create table noise.locations
  (                                     
	name varchar(100),
	complaint varchar(100), descript varchar(100),
	boro varchar(50),
	lat float8,
	lon float8,
	geom geometry(POINT, 4326)
  );
  ```

- inputing values for the geometry type after loading data from a CSV:  
`update noise.locations set the_geom = ST_SetSRID(ST_MakePoint(lon, lat), 4326);`

- adding a geometry column in a non-spatial table:  
  `select addgeometryColumn('table_name', 'geom', 4326, 'POINT', 2);`

- calculating area in EPSG 4326:  
  `alter table noise.hoods set area = (select ST_Area(geom::geography));`


### Common Spatial Queries
You may view more of these in [my intro to Visualizing Geospatial Data with CartoDB](https://github.com/clhenrick/cartodb-tutorial/tree/master/sql).

**Find all polygons from dataset A that intersect points from dataset B:**

```
SELECT a.*
FROM table_a_polygons a, table_b_points b
WHERE ST_Intersects(a.the_geom, b.the_geom);
```

**Find all rows in a polygon dataset that intersect a given point:**

```
-- note: geometry for point must be in the order lon, lat (x, y)
SELECT * FROM nyc_tenants_rights_service_areas
where
ST_Intersects(
  ST_GeomFromText(
   'Point(-73.982557 40.724435)', 4326
  ),
  nyc_tenants_rights_service_areas.the_geom    
);
```

Or using `ST_Contains`:

```
SELECT * FROM nyc_tenants_rights_service_areas
where
st_contains(
  nyc_tenants_rights_service_areas.the_geom,
  ST_GeomFromText(
   'Point(-73.917104 40.694827)', 4326
  )      
);
```

**Counting points inside a polygon:**

With ST_Containts():

```
SELECT us_counties.the_geom_webmercator,us_counties.cartodb_id,
count(quakes.the_geom)
AS total
FROM us_counties JOIN quakes
ON st_contains(us_counties.the_geom,quakes.the_geom)
GROUP BY us_counties.cartodb_id;
```

To update a column from table A with the number of points from table B that intersect table A's polygons:  

```
update noise.hoods set num_complaints = (
	select count(*)
	from noise.locations
	where
	ST_Intersects(
		noise.locations.geom,
		noise.hoods.geom
	)
);
```

**Select data within a bounding box**  
Using [`ST_MakeEnvelope`](http://postgis.refractions.net/docs/ST_MakeEnvelope.html)

HINT: You can use [bboxfinder.com](http://bboxfinder.com/) to easily grab coordinates
of a bounding box for a given area.

```
SELECT * FROM some_table
where geom && ST_MakeEnvelope(-73.913891, 40.873781, -73.907229, 40.878251, 4326)
```

**Make a line from a series of points**

```
SELECT ST_MakeLine (the_geom ORDER BY id ASC)
AS the_geom, route
FROM points_table
GROUP BY route;
```

**Order points in a table by distance to a given lat lon**  
This one uses CartoDB's built-in function `CDB_LatLng` which is short hand for doing:  
`SELECT ST_Transform( ST_GeomFromText( 'Point(-73.982557 40.724435)',),4326)`

```
SELECT * FROM table
ORDER BY the_geom <->
CDB_LatLng(42.5,-73) LIMIT 10;
```

**Access the previous row of data and get value (time, value, number, etc) difference**

```
WITH calc_duration AS (
 SELECT
 cartodb_id,
 extract(epoch FROM (date_time - lag(date_time,1) OVER(ORDER BY date_time))) AS
duration_in_seconds
 FROM tracking_eric
 ORDER BY date_time
)
UPDATE tracking_eric
SET duration_in_seconds = calc_duration.duration_in_seconds
FROM calc_duration
WHERE calc_duration.cartodb_id = tracking_eric.cartodb_id
```

**select population density by county**

In this one we cast the geometry data type to the geography data type to get units of measure in meters.

```
SELECT pop_sqkm,
 round( pop / (ST_Area(the_geom::geography)/1000000))
 as psqkm
 FROM us_counties
```


### Spatial Indexing
Makes queries hella fast. [OSGeo](http://revenant.ca/www/postgis/workshop/indexing.html) has a good tutorial.

- Basically the steps are:  
  `CREATE INDEX table_name_gix ON table_name USING GIST (geom);`  
  `VACUUM ANALYZE table_name`  
  `CLUSTER table_name USING table_name_gix;`  
  ***Do this every time after making changes to your dataset or importing new data.**

### Importing Spatial Data to PostGIS
#### Using shp2pgsql
1. Do:  
`shp2pgsql -I -s 4326 nyc-pediacities-hoods-v3-edit.shp noise.hoods > noise.sql`  
Or for using the geography data type do:  
`shp2pgsql -G -I nyc-pediacities-hoods-v3-edit.shp noise.nyc-pediacities-hoods-v3-edit_geographic > nyc_pediacities-hoods-v3-edit.sql`

2. Do:  
`psql -d nyc_noise -f noise.sql`  
Or for the geography type above:  
`psql -d nyc_noise -f nyc_pediacities-hoods-v3-edit.sql `

#### Using osm2pgsql
To import an OpenStreetMap extract in PBF format do:  
`osm2pgsql -H localhost --hstore-all -d nyc_from_osm ~/Downloads/newyorkcity.osm.pbf`

#### Using ogr2ogr
Example importing a GeoJSON file into a database called nyc_pluto:  

```
ogr2ogr -f PostgreSQL \
PG:"host='localhost' user='chrislhenrick' port='5432' \
dbname='nyc_pluto' password=''" \
bk_map_pluto_4326.json -nln bk_pluto
```


### Exporting Spatial Data from PostGIS
The two main tools used to export spatial data with more complex geometries from Postgres/PostGIS than points are `pgsql2shp` and `ogr2ogr`.

#### Using pgsql2shp
`pgsql2shp` is a tool that comes installed with PostGIS that allows for exporting data from a PostGIS database to a shapefile format. To use it you need to specify a file path to the output shapefile (just stating the basename with no extension will output in the current working directory), a host name (usually this is `localhost`), a user name, a password for the user, a database name, and an SQL query.

```
pgsql2shp -f <path to output shapefile> -h <hostname> -u <username> -P <password> databasename "<query>"
```

A sample export of a shapefile called `my_data` from a database called `my_db` looks like this:

```
pgsql2shp -f my_data -h localhost -u clhenrick -P 'mypassword' my_db "SELECT * FROM my_data "
```

#### Using ogr2ogr
**Note:** You may need to set the `GDAL_DATA` path if you git this error:

```
ERROR 4: Unable to open EPSG support file gcs.csv.
Try setting the GDAL_DATA environment variable to point to the
directory containing EPSG csv files.
```
If on Linux / Mac OS do this: `export GDAL_DATA=/usr/local/share/gdal`  
If on Windows do this: `C:\> set GDAL_DATA=C:\GDAL\data`

**To Export Data**  
Use ogr2ogr as follows to export a table (in this case a table called `dob_jobs_2014`) to a `GeoJSON` file (in this case a file called dob_jobs_2014_geocoded.geojson):

```
ogr2ogr -f GeoJSON -t_srs EPSG:4326 dob_jobs_2014_geocoded.geojson \
PG:"host='localhost' dbname='dob_jobs' user='chrislhenrick' password='' port='5432'" \
-sql "SELECT bbl, house, streetname, borough, jobtype, jobstatus, existheight, proposedheight, \
existoccupancy, proposedoccupany, horizontalenlrgmt, verticalenlrgmt, ownerbusinessname, \
ownerhousestreet, ownercitystatezip, ownerphone, jobdescription, geom \
FROM dob_jobs_2014 WHERE geom IS NOT NULL"
```

- **note:** you must select the column containing the geometry (usually `geom` or `wkb_geometry`) for your exported layer to have geometry data.

## Other Methods of Interacting With Postgres/PostGIS
to do...
### PGAdmin

### Python

### Node JS
