---
title: Vector Maps
subtitle: Introduction, Data Sources and Storage
tags: [gis, maps]

---

As [vector maps](https://en.wikipedia.org/wiki/Vector_tiles) have been really successful (see [maptiler example](https://www.maptiler.com/maps/#topo//vector/13.55/10.99274/47.45656)) in the last years I want to upgrade my own map and [toolchain](https://github.com/henrythasler/TileGenerator) to fancy new vector tiles technology. This series of blog posts serves mainly to sort out my thoughts and hopefully make life easier for the next guy to try this.

This is what the map looks like at the moment:

![rastermap](/img/blog/cyclemap-raster-current.png)

As the process of presenting a vector tile map to an end-user requires a whole set of tools (aka Toolchain), I will try to cover all parts of the design- and implementation-process.

All project-files will be stored on github.com/henrythasler/vectortiles. Create an issue there, if you have any questions, hints or whatevers.

## Goals

The following goals MUST be reached by the new toolchain/setup:

- A free source for spatial data (e.g. openstreetmap)
- Means of pre-processing and storing the spatial data (e.g. in a database)
- Content and quality of the resulting map is roughly the same as it is with the current toolchain.
- Use open source software whenever possible
- Dockerize the whole toolchain for portability
- support for high-DPI screens

## Stretch Goals

The following items would be nice-to-have

- Existing style definitions (CartoCSS, Mapnik XML) can be reused/converted
- Map design process is supported by some GUI/Editor
- Toolchain can also generate raster tiles for offline use with low-end devices (smartphone).

## Sifting the web for useful information

During my initial research I found these pages on various related topics helpful (random order):

- How to make mvt with PostGIS by Parlons Tech
- Vector tiles, PostGIS and OpenLayers by Giovanni Allegri
- awesome-vector-tiles by mapbox
- OpenMapTiles.org
- Using the new MVT function in PostGIS by Chris Whong
- Vector Tiles - Introduction & Usage with QGIS by Pirmin and Kalberer
- MVT generation: Mapnik vs PostGIS by Rafa de la Torre

## Sources for geospatial data

There are two main group of providers for geospatial data.

Companies or Organisations that provide geospatial-data and services based on [OpenStreetMap-data](https://wiki.openstreetmap.org/wiki/Main_Page) (osm):

- [Geofabrik](http://www.geofabrik.de/)
- [openmaptiles](https://openmaptiles.com/) 
- [maptiler](https://www.maptiler.com/)

Full-stack map-providers:

- [here](https://www.here.com/)
- [google](https://mapstyle.withgoogle.com/)

Almost all of them provide services where you can create your own map styles and share them via their website/toolchain. There are various plans to cover the costs according to your needs/traffic. You can also add your own data-sets (like pets-per-squarekilometer) to show on top of various basemaps.  This might be useful if you are just looking to provide some specific content on a basemap.

All services provide styling capabilities based on a pre-defined subset of map features (e.g. I haven't found playground-POIs on mapbox). You can upload additional data sources but that still involves pre-processing of raw data.

## Data storage

As mentioned before, you can use the map service providers to host your geospatial data. This requires specific pre-processing locally/cloud.

The following methods of storing data are available right now:

- cloud-based 
 - format depends on map service provider
- local 
 - raw-format (e.g. as PBF) 
 - database
 - (...)

## Design decisions

### Data Sources
1. Use up-to-date geospatial data for limited geographic areas (e.g. Europe) provided by [Geofabrik](https://download.geofabrik.de/).
2. Use planet-wide data (e.g. administrative boundaries) from [OpenStreetMapData](http://openstreetmapdata.com/)
3. Use global data from [planet.osm](https://wiki.openstreetmap.org/wiki/Planet.osm) to add some more planet-wide data if needed.  

### Data Storage
1. Geospatial data will be stored in a postgres/postgis database.
2. Raster data will be stored locally on disk

## Database setup

I use my own [postgis-container](https://github.com/henrythasler/docker/tree/master/postgis) with the latest versions and some useful extensions.

## Importing OSM-Data

Importing of geospatial data will be done with [Imposm 3](https://imposm.org/). It has some additional features over [OSM2pgsql](https://wiki.openstreetmap.org/wiki/Osm2pgsql) that might be useful:

 - Generalizing/Simplification of data to create low-resolution data from existing tables that can be used for small zoom levels.
 - regex-ing of content/names/whatever
 - custom SQL-Filter
 
Import with Imposm requires a mapping-file where you can define what features are imported. You can also implement some advanced filtering mechanism and even have 

The import script and additional files can be found [here](https://github.com/henrythasler/vectortiles/tree/master/imposm).

Import of a small sample pbf file (14MB) takes around 10s
