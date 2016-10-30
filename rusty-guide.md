Your SQL guide to getting the Semonkong analysis done according to Rusty. Again there are many ways to so do not take this as gospel.  

# Extract your data to an OSM DB

Download the latest extract from geofabrik and load it into my database `lesotho16`.

```
cd osm
wget http://download.geofabrik.de/africa/lesotho-latest.osm.pbf

osm2pgsql -c -d lesotho16 -K -H localhost -U colinbroderick --slim --number-processes 4 lesotho-latest.osm.pbf
```

## Next we should add in our area to a new database table for convience.

Download the boundary.

```
wget https://raw.githubusercontent.com/MapLesotho/Team-Semonkong/master/map.geojson
```

Use ogr2ogr to add it to a table called semo area. We also transform it to google mercator project so we don't have to reproject for our calculations.

```
/Applications/Postgres.app/Contents/Versions/latest/bin/ogr2ogr -f "PostgreSQL" PG:"dbname=lesotho16 user=colinbroderick" semonkong.geojson -nln semo_area -overwrite -a_srs EPSG:4326 -t_srs EPSG:900913
```

You might notice that there are more than one feature in our semo area table and that the table represents a multiple line strings. We would like to have a polygon so let's make a new table:

```{sql}
DROP TABLE IF EXISTS semo_area1;
CREATE TABLE semo_area1 (
    id SERIAL PRIMARY KEY,
    name text,
    the_geom geometry(Geometry,900913) NOT NULL
);

-- combine the existing line segments into one single polygon
INSERT INTO semo_area1(name, the_geom)
select 'semonkong large', ST_Transform(ST_MakePolygon(wkb_geometry),900913) as the_geom
    from semo_area
    where ogc_fid = 1;
```

## Add in Semonkong town boundary as a second boundary

You must know that we modified this file to include one property called name so we can have that to distinguish our areas in the database.

```
#wget the file....on github

ogr2ogr -f "PostgreSQL" PG:"dbname=lesotho16 user=colinbroderick" semonkong-inner.geojson -nln semo_area1 -append -a_srs EPSG:4326 -t_srs EPSG:900913

```

## Final prep step - let's cut out Semonkong into it's own tables

As you'll see from the list of tasks we're not interested in all the columns in each table, so let's be precise for once.

Start with the polygons, since we're **only** interested in buildings and landuses:

```
DROP TABLE semo_polys;
create table semo_polys as (
    select osm_id, building, landuse, admin_level, way_area, way
    from planet_osm_polygon as ol
    where ST_Intersects(
        ol.way, 
        (select ST_SetSRID(the_geom, 900913) from semo_area1
        where semo_area1.id = 1)
    ) 
    AND (building IS NOT NULL OR  landuse IS NOT NULL)
)


-- get the line cut out
drop table semo_lines;
create table semo_lines as (
    select osm_id, highway, bridge, waterway, way
    from planet_osm_line as ol
    where ST_Intersects(
        ol.way, 
        (select ST_SetSRID(the_geom, 900913) from semo_area1
        where semo_area1.id = 1)
    ) 
    AND (highway IS NOT NULL OR waterway IS NOT NULL)
)
```


```
SELECT landuse, 
    round((sum(ST_Area(way))/10000)::numeric, 3) as hectares 
from semo_polys 
where landuse IS NOT NULL 
GROUP BY landuse
```

```
-- filter out those within the semo-area-inner
SELECT ol.landuse, count(landuse), 
    round((sum(ST_Area(ol.way))/10000)::numeric, 3) as hectares 
from semo_polys as ol
where ol.landuse IS NOT NULL AND ST_Intersects(ol.way, (select the_geom from semo_area1 where semo_area1.id = 3))
GROUP BY landuse
```




```
SELECT highway, 
    round(SUM(ST_Length(way)/1000)::numeric, 2) as km 
from semo_lines
where highway IS NOT NULL 
GROUP BY highway
```

```
SELECT highway, 
    round(SUM(ST_Length(way)/1000)::numeric, 2) as km 
from semo_lines as ol
where highway IS NOT NULL AND ST_Intersects(ol.way, (select the_geom from semo_area1 where semo_area1.id = 3))
GROUP BY highway
```


```
SELECT building,
    count(building),
    round(sum(ST_Area(way))::numeric, 0) as total_sqm,
    round(AVG(ST_Area(way))::numeric, 0) as average_sqm
from semo_polys 
where building IS NOT NULL 
GROUP BY building
```

```
-- inner area
SELECT building,
    count(building),
    round(sum(ST_Area(way))::numeric, 0) as total_sqm,
    round(AVG(ST_Area(way))::numeric, 0) as average_sqm
from semo_polys as ol
where building IS NOT NULL 
AND ST_Intersects(ol.way, (select the_geom from semo_area1 where semo_area1.id = 3))
GROUP BY building
```
## Results for large area

**landuse**
| type           | count | area (ha) | 
|----------------|-------|-----------| 
| "cemetery"     | 4     | 1.118     | 
| "quarry"       | 2     | 0.836     | 
| "industrial"   | 1     | 2.557     | 
| "farmland"     | 2476  | 12299.531 | 
| "farmyard"     | 33    | 1.900     | 
| "construction" | 1     | 1.887     | 
| "residential"  | 443   | 2170.163  | 
| "forest"       | 28    | 61.253    | 

**Roads (km)**
| type           | length (km) | 
|----------------|-------------| 
| "unclassified" | 25.41       | 
| "secondary"    | 49.87       | 
| "track"        | 507.95      | 
| "service"      | 4.95        | 
| "path"         | 672.86      | 
| "tertiary"     | 114.14      | 
| "construction" | 0.02        | 
| "residential"  | 131.33      | 


**Buildings:**

| type           | count | total_sqm | average_sqm | 
|----------------|-------|-----------|-------------| 
| "house"        | 222   | 12490     | 56          | 
| "shed"         | 12    | 306       | 25          | 
| "ruin"         | 2     | 45        | 22          | 
| "ruins"        | 137   | 3667      | 27          | 
| "yes"          | 3994  | 315141    | 79          | 
| "construction" | 267   | 16603     | 62          | 
| "hut"          | 6968  | 246791    | 35          | 
| "residential"  | 6     | 444       | 74          | 
| "huts"         | 4     | 74        | 18          | 
| "industrial"   | 1     | 328       | 328         | 


## Results for Inner Area

**Landuse (ha)**

| type           | area     | count | 
|----------------|---------:|------:| 
| "cemetery"     | 0.769    | 2     | 
| "forest"       | 31.220   | 2     | 
| "farmland"     | 1264.650 | 191   | 
| "farmyard"     | 0.999    | 2     | 
| "construction" | 1.887    | 1     | 
| "residential"  | 547.097  | 23    | 



**Roads (km)**

| type           | length (km) | 
|----------------|------------:| 
| "secondary"    | 14.50       | 
| "unclassified" | 9.42        | 
| "track"        | 75.32       | 
| "service"      | 0.28        | 
| "path"         | 26.78       | 
| "tertiary"     | 11.40       | 
| "residential"  | 47.98       | 


**Buildings:**

| type           | count | total_sqm | average_sqm | 
|----------------|------:|----------:|------------:| 
| "shed"         | 11    | 251       | 23          | 
| "house"        | 1     | 56        | 56          | 
| "industrial"   | 1     | 328       | 328         | 
| "yes"          | 1834  | 180446    | 98          | 
| "construction" | 95    | 9336      | 98          | 
| "hut"          | 436   | 14043     | 32          | 
| "residential"  | 2     | 255       | 127         | 

