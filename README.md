
https://stac.sogeo.services/files/raster/ch.so.agi.lidar_2019.ndsm_buildings.tif


```
docker run --rm --name edit-db -p 54321:5432 -v pgdata:/var/lib/postgresql/data:delegated -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=edit sogis/postgis:16-3.4
```

```
ALTER DATABASE edit SET postgis.enable_outdb_rasters = true;
ALTER DATABASE edit SET postgis.gdal_enabled_drivers TO 'ENABLE_ALL';

CREATE SCHEMA agi_lidar_2019_ndsm;
```

```
docker exec -it edit-db bash
```

```
raster2pgsql -s 2056 -t 512x512 -I -R /vsicurl/https://stac.sogeo.services/files/raster/ch.so.agi.lidar_2019.ndsm_buildings.tif agi_lidar_2019_ndsm.buildings | psql -h localhost -p 5432 -U postgres -d edit
```

```
SELECT 
    (ST_MetaData(rast)).* 
FROM 
        agi_lidar_2019_ndsm.buildings  
LIMIT 1;
```

```
SELECT 
    (ST_BandMetaData(rast)).* 
FROM 
        agi_lidar_2019_ndsm.buildings  
LIMIT 1;
```


```
java -jar /Users/stefan/apps/ili2pg-5.0.1/ili2pg-5.0.1.jar --dbhost localhost --dbport 54321 --dbusr postgres --dbpwd secret --dbdatabase edit --disableValidation --strokeArcs --defaultSrsCode 2056 --createGeomIdx --createFkIdx --createFk --nameByTopic --createEnumTabs --models DM01AVSO24LV95 --dbschema agi_dm01avsoch24 --doSchemaImport  --import ilidata:2517.ch.so.agi.av.dm01_so
```


```
UPDATE 
    agi_lidar_2019_ndsm.buildings
SET 
    rast = ST_SetBandNoDataValue(rast,1, -9999)
;
```


```
WITH buildings AS 
(
    SELECT 
        t_id,
        geometrie AS geometrie
    FROM 
        agi_dm01avsoch24.bodenbedeckung_boflaeche 
    WHERE 
        art = 'Gebaeude'
    --AND     
        --t_id = 2260
        --t_id = 3967
)
,
clipped_tiles AS 
(
  SELECT 
    ST_Clip(elev.rast, buildings.geometrie) AS rast, 
    elev.rid,
    buildings.t_id
  FROM 
    agi_lidar_2019_ndsm.buildings AS elev
  JOIN 
    buildings
    ON ST_Intersects(buildings.geometrie, ST_ConvexHull(elev.rast))
)
SELECT 
    t_id,
    (ST_SummaryStatsAgg(rast, 1, true)).*
FROM 
    clipped_tiles
GROUP BY
    t_id
;
```


```
WITH buildings AS 
(
    SELECT 
        t_id,
        geometrie AS geometrie
    FROM 
        agi_dm01avsoch24.bodenbedeckung_boflaeche 
    WHERE 
        art = 'Gebaeude'
    AND     
        t_id = 2260
        --t_id = 3967
)
,
clipped_tiles AS 
(
  SELECT 
    ST_Clip(elev.rast, buildings.geometrie) AS rast, 
    elev.rid,
    buildings.t_id
  FROM 
    agi_lidar_2019_ndsm.buildings AS elev
  JOIN 
    buildings
    ON ST_Intersects(buildings.geometrie, ST_ConvexHull(elev.rast))
)
SELECT 
    ST_AsGDALRaster(ST_Union(ST_Rescale(rast, 2.0)), 'AAIGrid') As rastaaigrid
FROM 
    clipped_tiles
;

```