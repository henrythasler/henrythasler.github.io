---
title: Serving Vector Tiles
subtitle: Evaluating various tools
tags: [gis, maps]

---

There are several tools available for serving vector tiles. Let's check the following:

- [T-Rex](https://t-rex.tileserver.ch/)
- [Tegola](https://tegola.io/)

### T-Rex

You need a running postgis-instance and need to specify the table that is being used:

Starting T-Rex requires just a Docker container:

{% highlight shell linenos %}
docker run --rm -ti --net gis -p 6767:6767 sourcepole/t-rex serve --dbconn postgresql://postgres@postgis/vector --bind=0.0.0.0
{% endhighlight %}

![](/img/blog/Selection_024.png)

Visiting [localhost:6767/#](http://localhost:6767/#) will show a simple user interface with the tables we imported:

![](/img/blog/Selection_025.png)

You can even view the data on a map by clicking the different buttons in the top-right corner:

![](/img/blog/Selection_026.png)

So far so good, BUT. Clicking on our buildings-table leads to a ton of errors in the t-rex console:

{% highlight shell linenos %}
2018-12-30 14:08:00.410 ERROR Layer 'osm_buildings': Unknown geometry type GEOMETRY
2018-12-30 14:08:00.410 ERROR Row { statement: StatementInfo { name: "s18", param_types: [Type(Float8), Type(Float8), Type(Float8), Type(Float8)], columns: [Column { name: "geometry", type_: Type(Other(Other { name: "geometry", oid: 50960, kind: Simple, schema: "public" })) }, Column { name: "id", type_: Type(Int4) }, Column { name: "osm_id", type_: Type(Int8) }, Column { name: "name", type_: Type(Varchar) }, Column { name: "type", type_: Type(Varchar) }] } }
{% endhighlight %}

A hint why this is the case is shown right after startup of the t-rex container:

{% highlight shell linenos %}
2018-12-30 14:10:15.540 WARN Multiple geometry types in "import"."osm_buildings".geometry: MULTIPOLYGON, POLYGON
{% endhighlight %}

It seems that T-Rex can't handle multiple geometry types. I was able to lessen the errors with a specific [config file](https://github.com/henrythasler/vectortiles/blob/master/t-rex/config.toml) that uses ST_MAKEVALID(geometry) to fix some invalid geometries but was yet unable to mitigate the issue altogether.

Visiting the roads-table was even worse:
![](/img/blog/Selection_027.png)
This was accompanied by database-restarts and complete freezing of my computer... Definitely NOT what we want. 

### Tegola

Setting up Tegola is a bit more work as it does not provide any means to view the data it serves. Tegola itself needs a config-file with layer-definitions similar to T-Rex. I roughly followed the [Tegola Mapbox Tutorial](https://tegola.io/tutorials/tegola-with-mapbox/) to setup a basic html-viewer for our map data.

You can find the required files [here](https://github.com/henrythasler/vectortiles/tree/master/tegola)

After starting up the tegola-server with `serve.sh` and loading the index.html page in a browser we get the following page:
![](/img/blog/Selection_028.png)

Zooming in a bit more shows all buildings as defined in Tegola-config (min_zoom=13):
![](/img/blog/Selection_029.png)

Tegola did not crash or show any other unexpected behavior during my tests. It has caching and simplification features and the queries can be defined in detail for each layer via [SQL-Property](https://tegola.io/documentation/configuration/#postgis-1). 

When zooming out a lot (z=5), the time to generate the tiles increases dramatically as the whole database will be dumped into one vector tile, so great care must be taken to optimize the amount of data for each zoom level with [generalized tables](https://imposm.org/docs/imposm3/latest/mapping.html#generalized-tables).

## Conclusion

Tegola it is for now.
