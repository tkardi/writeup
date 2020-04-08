---
title: "On writing GeoServer SLDs (part I)"
date: 2020-03-20T09:27:00+02:00
draft: false
---

Right. So some time ago I needed to do a bunch of SLDs for use with
[GeoServer](http://geoserver.org) with categorical classes: every polygon's fill
colored with a different color. The easy solution was to style the layer in
e.g. [QGIS](https://qgis.org) export the syle as SLD and use that with GeoServer.
It would have been just this trivial a task but as always there were a couple of
_gotchas_:

- the number of different layers needing portrayal was _unknown_,
- the number of classes for at least one of the layers was enormous (>9K), and
- the classes change over time: some disappear and new ones are added constantly

I knew previously about [_attribute-based styling_](https://docs.geoserver.org/stable/en/user/styling/sld/cookbook/polygons.html#attribute-based-polygon)
where you define a `FeatureTypeStyle/Rule` with an associated `Filter` and a
`PolygonSymbolizer` for this specific rule. Neat. But if you need to cook up
filters for ump-teen different layers and then possibly someone changes
something in any one of the codelist data that you were filtering your styles
by ... so no, this wouldn't do.

But as the layers at hand were already
[SQL view](https://docs.geoserver.org/stable/en/user/data/database/sqlview.html)-
based, it was totally feasible to retrive the hex color code with the data from the
database either precalculated or calculated per request. So what
I was looking for was the capability of doing _random coloring_ for polygons
using SLD... except the colors had to be constant per value for every WMS GetMap
request and not change in time... unless they needed to change.

To recap, essentially: a polygon's fill can be of any random color but it needs
to be constant in time based on whatever value that is chosen.

The solution that was finally adopted consisted of calculating a `md5`
hash of the property that you want to color your polygons by and then slice 6
chars this from this 32-char string for your hex color code in the GeoServer
SQL view.

The only downside of this approach is that the WMS GetLegendGraphic requests
will not return anything sensible (because the legend will be data-dependent).
And even if it did who would want to see a 9K color-entry legend. There's also
a risk of neighboring areas being assigned the same-like colors but currently
for the workflow needed this poses no problems. But it should be kept in mind
still...

But right, the cool thing is - if you name this _hex-color-column_ in your views
by the same name you make it possible to reuse the style instantly.

But enough talk, lets do a run-through of this. You are welcome to follow along
with your own dataset but if there's nothing interesting you can find then
lets download the Estonian administrative units
[settlements layer](https://geoportaal.maaamet.ee/docs/haldus_asustus/asustusyksus_shp.zip) from
[the Estonian Land Board's](https://geoportaal.maaamet.ee/eng/Spatial-Data/Administrative-and-Settlement-Division-p312.html)
homepage and i'll import that to my local PostGIS database using shp2pgsql.

```
~/data/tmp$ wget https://geoportaal.maaamet.ee/docs/haldus_asustus/asustusyksus_shp.zip -O asustusyksus_shp.zip
 [..]
~/data/tmp$ unzip asustusyksus_shp.zip
Archive:  asustusyksus_shp.zip
  inflating: asustusyksus_20200301.dbf  
  inflating: asustusyksus_20200301.prj  
  inflating: asustusyksus_20200301.shp  
  inflating: asustusyksus_20200301.shx
~/data/tmp$ shp2pgsql -d -s 3301 -g geom -W "cp1257" -I  asustusyksus_20200301.shp public.asustusyksus | psql -h localhost -d postgres -U postgres --quiet
 [..]
~/data/tmp$
```

Next in the GeoServer web ui set up a new [SQL view](https://docs.geoserver.org/stable/en/user/data/database/sqlview.html)-
based layer with the following query (in a workspace called `public` using a
PostGIS datastore)

```
select
    gid, animi as settlement_name, akood as settlement_code,
    tyyp as settlement_type, onimi as municipality_name, okood as municipality_code,  
    mnimi as county_name, mkood as county_code, geom
from
    public.asustusyksus
```
and publish it with the default settings. Now browsing the
[layer preview at your local GeoServer](http://localhost:8080/geoserver/public/wms?service=WMS&version=1.1.0&request=GetMap&layers=public%3Av_settlements&bbox=448183.07559659635%2C6435664.640892181%2C723048.3349062635%2C6557826.978363144&width=1800&height=800&srs=EPSG%3A3301&format=application/openlayers) yields something in the line of

![Settlement units WMS with all defaults](../img/settlements_1.png)

Right, this is nice and grayish. In order to bring some colors into play here
we'll need to do two things:

- change SQL view query so it outputs a hex color code based on some column,
say `municipality_name`
(in order to get a patchy and colorful map of Estonian settlements colored by
municipality) into a column called `_hex`
- add a SLD to style the polygons based on `_hex`.

Note: I'm prefixing this computed column name with an underscore so for example
for supporting GetFeatureInfo requests this column can easily be filtered out
from the response.

- Note 2: I'm using `municipality_name` not `municipality_code` here because
municipality code is a statistic codelist value. So if a municipality gets a
new official identifier (although the name stays the same) because of whatever
reasons then we'd get a new color. But using the name helps us to retain the
color (unless the name changes aswell of course).

So, open up the GeoServer web ui SQL view editor again and change the query to

```
select
    gid, animi as settlement_name, akood as settlement_code,
    tyyp as settlement_type, onimi as municipality_name, okood as municipality_code,  
    mnimi as county_name, mkood as county_code, geom,
    '#'||right(md5(onimi),6) as _hex
from
    public.asustusyksus
```

Now for the SLD style, create a new style. I'm calling it `area.autostyle.hex`.
Otherwise it is essentially the same as any other polygon `FeatureTypeStyle` with the only
difference that we're using

```
<ogc:PropertyName>_hex</ogc:PropertyName>
```
as the input to PolygonSymbolizer's fill

```
[..]
<FeatureTypeStyle>
  <Rule>
    <PolygonSymbolizer>
      <Fill>
        <CssParameter name="fill">
          <ogc:PropertyName>_hex</ogc:PropertyName>
        </CssParameter>
        <CssParameter name="fill-opacity">0.8</CssParameter>
      </Fill>
      <Stroke>
        <CssParameter name="stroke">#030303</CssParameter>
        <CssParameter name="stroke-width">0.1</CssParameter>
      </Stroke>
    </PolygonSymbolizer>
  </Rule>
</FeatureTypeStyle>
[..]
```

The full SLD file is available
[here](https://github.com/tkardi/sld-diary/blob/master/src/data/workspaces/public/styles/area.autostyle.hex.sld)

Don't forget to change the layer's default style to the new `area.autostyle.hex`
aswell. Now browsing again the
[layer preview at your local GeoServer](http://localhost:8080/geoserver/public/wms?service=WMS&version=1.1.0&request=GetMap&layers=public%3Av_settlements&bbox=448183.07559659635%2C6435664.640892181%2C723048.3349062635%2C6557826.978363144&width=1800&height=800&srs=EPSG%3A3301&format=application/openlayers) yields something in the line of

![Settlement units colored by municipality name](../img/settlements_hex_municipality.png)

All required `$GEOSERVER_DATA_DIR/workspaces` files to set this up locally are available at
[this github repo](https://github.com/tkardi/sld-diary/tree/master/src/data/workspaces)
under [CC-BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

Next time around I'll write something on how to compose a SLD file so you would never
have compose any more SLDs again.
