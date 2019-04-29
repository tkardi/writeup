---
title: "Building a flightradar in Leaflet (part II)"
date: 2019-04-29T04:42:00+02:00
draft: false
---
{{< load-leaflet >}}

This post will pick up where we left off [last time](
../../27/building-a-totally-useless-map/) with building a radar-like
map in [Leaflet](https://leafletjs.com/) with data from the
[OpenSky Network](https://opensky-network.org/)'s public [API](
https://opensky-network.org/apidoc/).

Last time we finished with Leaflet map that displays real-time aircraft
locations using a local proxy for transforming the OpenSky's public API
response JSON to GeoJSON

{{< leaflet-realtime
    id="1"
    height="350px" width="100%"
    realtimeURL="https://tkardi.ee/current/flightradar/?format=json" >}}

## Building a radar-like UI
As summarised last time, we'll need to add a polar graticule with a 360-degrees
revolving _hand_ and then work on the visibility effects of the aircraft icons.
We had the map center defined as

{{< highlight javascript >}}
// set up map
var center = [58.65, 25.06];
{{</ highlight >}}

just underneath it, add another variable definition

{{< highlight javascript >}}
var radarbeam = {
    "type":"LineString",
    "coordinates": [
        [center[1], center[0]],
        [center[1], center[0]]
    ]};
{{</ highlight >}}

from this GeoJSON we'll construct both the radar graticule and the revolving
hand.

### Graticule
For the graticule we'll simply draw a set of concentric circles spaced evenly
at 50K of those Web-Mercator units, at 50000, 100000, 150000, 200000, and 250000.

{{< highlight javascript >}}
// draw radar's "bulls-eye"
// with alternating border thickness
var rings = [];
[50000, 100000, 150000, 200000, 250000].forEach(function(r) {
    rings.push(
        L.circle(
            center, {
                radius: r,
                fill: false,
                weight: r % 100000 == 0 ? 1.75 : 0.75,
                color: '#808080'
            }
        ).addTo(map)
    );
});
{{</ highlight >}}

Another thing we'll need are the axis for angle demarcation - North (0°), East
(90°), S (180°), W (270°) as majors and the minor ones in between. The WGS84
coordinates that the data is in do not work well with straight forward
Cartesian coordinate arithmetic, so we'll need to jump some hoops.

First, we'll need to get the projected coordinates of the map's center
{{< highlight javascript >}}
var xy1 = map.options.crs.project(L.latLng(center));
{{</ highlight >}}

And then calculate the endpoints of two lines (with an arbitrary) length but
crossing each other at right angles through our map center.

{{< highlight javascript >}}
var radius = 550000;
var right = L.point(xy1).add([radius, 0]),
    left = L.point(xy1).subtract([radius, 0]),
    top = L.point(xy1).add([0, radius]),
    bottom = L.point(xy1).subtract([0, radius]);

var crosshairs = [
    L.polyline(
        [map.options.crs.unproject(left), map.options.crs.unproject(right)],
        {weight: 1.75, color: '#808080'}
    ).addTo(map),
    L.polyline(
        [map.options.crs.unproject(top), map.options.crs.unproject(bottom)],
        {weight: 1.75, color: '#808080'}
    ).addTo(map)
];
{{</ highlight >}}

For the minor axis we'll calculate four separate lines and not let them
cross the center point.

{{< highlight javascript >}}
[45, 135, 225, 315].forEach(function(angle) {
    crosshairs.push(
        L.polyline([
            map.options.crs.unproject(L.point([
                xy1.x + Math.sin(angle * Math.PI / 180) * 75000,
                xy1.y + Math.cos(angle * Math.PI / 180) * 75000
            ])),
            map.options.crs.unproject(L.point([
                xy1.x + Math.sin(angle * Math.PI / 180) * radius,
                xy1.y + Math.cos(angle * Math.PI / 180) * radius
            ]))
        ],
            {weight: 0.75, color: '#808080'}
        ).addTo(map)
    );
});
{{</ highlight >}}

And as a final touch - add angle labels to the tips of major axis. For that
we'll simply draw 0-length `L.polyline`s to the tips of the major axis and then
label them with _permanent tooltips_.


{{< highlight javascript >}}
var anglelabels = [
    L.polyline(
        [map.options.crs.unproject(left), map.options.crs.unproject(left)],
        {weight: 0.1, color: '#fffff', opacity:0}
    ).addTo(map).bindTooltip(
        '<b>270° </b>',
        {permanent: true, opacity: 0.7, direction: 'left'}
    ).openTooltip(),
    L.polyline(
        [map.options.crs.unproject(right), map.options.crs.unproject(right)],
        {weight: 0.1, color: '#fffff', opacity:0}
    ).addTo(map).bindTooltip(
        '<b> 90°</b>',
        {permanent: true, opacity: 0.7, direction: 'right'}
    ).openTooltip(),
    L.polyline(
        [map.options.crs.unproject(top), map.options.crs.unproject(top)],
        {weight: 0.1, color: '#fffff', opacity:0}
    ).addTo(map).bindTooltip(
        '<b>0°</b>',
        {permanent: true, opacity: 0.7, direction: 'top'}
    ).openTooltip(),
    L.polyline(
        [map.options.crs.unproject(bottom), map.options.crs.unproject(bottom)],
        {weight: 0.1, color: '#fffff', opacity:0}
    ).addTo(map).bindTooltip(
        '<b>180°</b>',
        {permanent: true, opacity: 0.7, direction: 'bottom'}
    ).openTooltip()
];
{{</ highlight >}}

And now we'll have this kind of view for a map

{{< leaflet-realtime
    id="2"
    height="350px" width="100%"
    addCraticule="true" >}}

### A revolving hand
As a final piece for the radar-like UI let's add the 360-degree revolving
hand. For that we'll simply animate a linestring, recalculating its end
coordinate in an interval suitable to leave a smooth animated feeling. In
addition, let's construct a sector of a "radar beam", that can later be used
for finding out which aircraft icons are/were currently (or a moment ago)
under the radar swath. The variable `radarbeam` was defined already before,
so:

{{< highlight javascript >}}
var radar = L.geoJSON(
    radarbeam, {
        onEachFeature : function(feature, layer) {
            var arclength = 2;
            var sumangle = 360;

            // sector is the slice of circle we'll use as a "beam shadow"
            // aswell use it to test point-in-polygon for aircraft icon fade-out
            var sector = {
                type:"Polygon",
                coordinates: [ [
                    feature.coordinates[0], feature.coordinates[1],
                    feature.coordinates[1], feature.coordinates[0]
                ] ]
            };

            var beamshadow = L.geoJSON(
                sector, {
                    style: function(feature){
                        // use an extra-classname if any special styling
                        // needs are required
                        return {
                            opacity:0.75,
                            color: '#109856',
                            weight:0.2,
                            className:'radar-hand'
                        }
                    }
                }
            ).addTo(map);

            setInterval(function(){
                // animate "radar beam"
                if (sumangle >= 360) {
                    sumangle = 0;
                } else {
                    sumangle += arclength;
                }
                var beamlatlngs = layer.getLatLngs(),
                    beamshadowlatlngs = beamshadow.getLayers()[0].getLatLngs();

                // calculate a new location for the beam linestring.

                beamlatlngs[1] = map.options.crs.unproject(
                    L.point([
                        xy1.x + Math.sin(sumangle * Math.PI / 180) * radius,
                        xy1.y + Math.cos(sumangle * Math.PI / 180) * radius
                    ])
                );

                // and a new location for the trailing corner of the beam shadow

                beamshadowlatlngs[0][1] = map.options.crs.unproject(
                    L.point([
                        xy1.x +
                            Math.sin(
                                (sumangle-5*arclength) * Math.PI/180
                            ) * radius,
                        xy1.y +
                            Math.cos(
                                (sumangle-5*arclength) * Math.PI/180
                            ) * radius
                    ])
                );

                var next = [
                    beamshadowlatlngs[0][0], beamlatlngs[1],
                    beamshadowlatlngs[0][1], beamshadowlatlngs[0][0]
                ];
                beamshadow.getLayers()[0].setLatLngs(next);
                layer.setLatLngs(beamlatlngs).bringToFront();
            }, 50);
        },
        style: function(feature) {
            return {color: '#109856', weight: 3, opacity:0.5}
        }
    }
).addTo(map);
{{</ highlight >}}

In addition we can add a `svg` blur to the `beamshadow` layer (remember the
extra classname, `radar-hand`). Add to the `style` tag of the html doc:

{{< highlight css >}}
.radar-hand {
    filter: url(#blur);
}
{{</ highlight >}}

and then a `svg` blur to the `html` doc, e.g. into the map `div`:

{{< highlight html >}}
<div id="map">
    <svg height="140" width="140">
        <defs>
            <filter id="blur" x="0" y="0" width="100%" height="100%">
                <feGaussianBlur
                    result="blurOut"
                    in="SourceGraphic"
                    stdDeviation="5"/>
            </filter>
        </defs>
    </svg>
</div>
{{</ highlight >}}

And pulling this all together should yield something in the line of

{{< leaflet-realtime
    id="3"
    height="350px" width="100%"
    addCraticule="true"
    addHand="true" >}}

### Fade-out effects for aircraft icons

Here we'll have to deal with a couple of things. We'll want to show the
aircraft icon only when the _radar beam_ has passed over it and then after some
seconds let it fade out (animate its opacity to 0). The same with the tooltips -
these should be opened automatically and then closed when the icon fades out
in order to make room for other aircraft icons and other labels because
otherwise there will be too much clutter.

One of the ways to deal with the fadeout is in css. From before we had this
declaration in styles

{{< highlight css >}}
.aeroplane-visible {
    background: #109856;
    border: none;
    opacity: 1.0;
}
{{</ highlight >}}

Now lets had a `end` class to this aswell, with say after 5 seconds fade
the opacity of the icon from the starting `1` to `0.01`. And in addition
we're going to have to change the style that the inbound aircraft icons get,
let's simply class it as `.aeroplane`.

{{< highlight css >}}
.aeroplane {
    opacity: 0
}
.aeroplane-visible.end{
    transition: opacity 5s ease-in-out;
    opacity: 0.01;
}
{{</ highlight >}}

First off lets correct the classname that the inbound data icons get, from
`aeroplane-visible` to `aeroplane`.

{{< highlight javascript >}}
pointToLayer: function(feature, latlng) {
    var marker = L.marker(latlng, {
        icon: L.divIcon({
            className:'aeroplane', // <-- this line must be changed
            iconSize: [10,10]
        }),
        riseOnHover: true
    }).bindTooltip(
        '<b>{callsign}</b><br>Alt: {geo_altitude} m @ {velocity} m/s'.replace(
            L.Util.templateRe, function (str, key) {
                var value = feature.properties[key];
                if (value === undefined || value == null) {
                    value = 'N/A';
                }
                return value;
            }),
        {
            permanent: false, opacity: 0.7}
    );
    return marker;
},
{{</ highlight >}}


Now all that is needed to attach the class at the right moment in time. The
correct moment would be in the radar-hand animation, in the end of
`radar.options.setInterval` function

{{< highlight javascript >}}
realtime.getLayers().forEach(function(layer){
    var latlng = layer.getLatLng(),
        el = layer.getElement();

    // test if the ircraft location is within the beam-shadow sector
    if (realtime.options.pointInPolygon(latlng, next)) {
        // remove fade-out if it exists
        L.DomUtil.removeClass(el, 'end');

        // make icon visible
        L.DomUtil.addClass(el, 'aeroplane-visible');

        // and set animation for tooltip
        layer.openTooltip();
        setTimeout(function(){
            layer.closeTooltip();
        }, 5000);
    } else {
        // otherwise start fade-out
        // this should kick in only if the classname is
        // not already there
        L.DomUtil.addClass(el, 'end');
    }
});
{{</ highlight >}}

But in order to use this we'll have to test for the points containment within
the radar beam shadow sector thing we added. This could be done in Leaflet using
the longitude/latitude bounds of sector but this will give us horrific results.
After scouring the internets I found a working implementation for the ray-casting
algorithm in JavaScript from
[substack/point-in-polygon](https://github.com/substack/point-in-polygon).
This work is licensed under
[the MIT License](https://github.com/substack/point-in-polygon/blob/master/LICENSE).
With a few retouches for simpler use with a Leaflet layer:

{{< highlight javascript >}}
pointInPolygon: function (latlng, latlngs) {
    // with a slight modifications taken from
    // https://github.com/substack/point-in-polygon/blob/master/index.js
    // which is licensed under the MIT license
    // https://github.com/substack/point-in-polygon/blob/master/LICENSE
    var x = latlng.lng, y = latlng.lat;

    var inside = false;
    for (var i = 0, j = latlngs.length - 1; i < latlngs.length; j = i++) {
        var xi = latlngs[i].lng, yi = latlngs[i].lat;
        var xj = latlngs[j].lng, yj = latlngs[j].lat;

        var intersect = ((yi > y) != (yj > y))
            && (x < (xj - xi) * (y - yi) / (yj - yi) + xi);
        if (intersect) inside = !inside;
    }
    return inside;
}
{{</ highlight >}}

We'll add this simply to our `realtime.options` and now we should all settled.

{{< leaflet-realtime
    id="4"
    height="350px" width="100%"
    realtimeURL="https://tkardi.ee/current/flightradar/?format=json"
    addCraticule="true"
    addHand="true" >}}

## Summary
The full code for the HTML doc is also available as a
[gist](https://gist.github.com/tkardi/99fb0a96b12068157af30cf806651bcf). If
you have any questions or comments please feel free to contact me.
