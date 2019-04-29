---
title: "Building a flightradar in Leaflet (part I)"
date: 2019-04-27T04:42:00+02:00
draft: false
---
{{< load-leaflet >}}

This post is about building a radar-like map in [Leaflet](https://leafletjs.com/)
with data from the [OpenSky Network](https://opensky-network.org/)'s public
[API](https://opensky-network.org/apidoc/).

It's a quick one-evening hack so don't expect much _finesse_. The main motivation
behind it being just to prove myself that I can do it.

## API
The OpenSky Network provides a public API for retrieving live airspace
information for research and non-commercial purposes. The API documentation
is available [here](https://opensky-network.org/apidoc/). Instead of using it
directly I set up a proxy which in turn formats the response data to GeoJSON.
The proxy itself is fairly simple and written in Python. Use `requests` to
retrieve the data and then reformat everything to GeoJSON like this:

{{< highlight python >}}
import requests
from collections import OrderedDict

# note the use of bounding box coords in the URL
# this is roughly the area for Latvia, Estonia and
# a small part of southern Finland
url = 'https://opensky-network.org/api/states/all?lamin=57.48&lomin=21.6&lamax=59.82&lomax=28.52'
keys = [
    'icao24', 'callsign', 'origin_country', 'time_position', 'last_contact',
    'longitude', 'latitude', 'baro_altitude', 'on_ground', 'velocity',
    'true_track', 'vertical_rate', 'sensors', 'geo_altitude', 'squawk',
    'spi', 'position_source'
]

def get_flight_radar_data():
    r = requests.get(url)
    r.raise_for_status()
    return to_geojson(r.json())

def to_geojson(data):
    f = [
        OrderedDict(
            type='Feature',
            id=ac[0],
            geometry=OrderedDict(type='Point', coordinates=[ac[5],ac[6]]),
            properties=OrderedDict(zip(keys, ac))
        ) for ac in data.get('states', [])
    ]

    return dict(
        type='FeatureCollection',
        features=f
    )
{{</ highlight >}}

Saved this as `proxy.py` and then used my existing Django install on a shared
hosting site to run it. Otherwise it can be run on localhost as a Flask app with

{{< highlight python >}}
import json
from flask import Flask
from flask import Response

from proxy import get_flight_radar_data

app = Flask(__name__)

@app.route('/flightradar')
def flightradar():
    return Response(
        response=json.dumps(get_flight_radar_data()),
        status=200,
        mimetype='application/json'
    )
{{</ highlight >}}

Save this as `flaskapp.py` in the same path as the previous `flightradar.py`,
and then fire it up:

{{< highlight bash >}}
$ export FLASK_APP=flaskapp.py
$ flask run
 * Serving Flask app "flaskapp.py"
 * Environment: production
   WARNING: Do not use the development server in a production environment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
{{</ highlight >}}

So now the GeoJSON enabled API is ready for use and the data can be viewed
using other applications (e.g. QGIS) aswell.


## Animating (realtime) markers on a Leaflet map
There's the excellent [Leaflet.Realtime](https://github.com/perliedman/leaflet-realtime)
library for _putting realtime data on a Leaflet map_. This will help us to
query the API we just set up after a preset interval and will manage all
the necessary redraws etc.

### Set up Leaflet map
Let's start off by setting up the Leaflet map using the Stamen Toner basemap.
In your favourite text-editor:

{{< highlight html >}}
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>Air traffic map</title>
        <link
            rel="stylesheet"
            type="text/css"
            href="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.0.1/leaflet.css"/>
        <script
            src="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.0.1/leaflet.js"
            type="text/javascript">
        </script>
        <style>
            #map {
                position: absolute;
                top: 0;
                left: 0;
                bottom: 0;
                right: 0;
            }
        </style>
    </head>
    <body>
        <div id="map">
        </div>
        <script>
            // set up map
            var center = [58.65, 25.06];
            var map = L.map('map').setView(center, 7);

            // Stamen's Toner basemap
            L.tileLayer(
                'https://stamen-tiles-{s}.a.ssl.fastly.net/toner/{z}/{x}/{y}.png', {
                    attribution: 'Map tiles by <a href="http://stamen.com">' +
                        'Stamen Design</a>, under' +
                        '<a href="http://creativecommons.org/licenses/by/3.0">' +
                        'CC BY 3.0</a>. Data by <a href="http://openstreetmap.org">' +
                        'OpenStreetMap</a>, under' +
                        '<a href="http://www.openstreetmap.org/copyright">ODbL</a>.'
            }).addTo(map);
        </script>
    </body>
</html>
{{</ highlight >}}

Save this as `flightradar.html` and open the file in your web-browser.

### Animate the data
Now in order to use `Leaflet.Realtime` to do the data querying and
draw/animation, add

{{< highlight html >}}
<script
    src="https://cdnjs.cloudflare.com/ajax/libs/leaflet-realtime/2.0.0/leaflet-realtime.min.js"
    type="text/javascript">
</script>
{{</ highlight >}}

to the `head` section of your `flightradar.html` file. Next work a bit on the
`body`'s `script` tag and add

{{< highlight javascript >}}
// air traffic locations layer,
// should actually request when the "radar beam" is pointing N e.g
// but we'll go with L.Realtime for the moment.
var realtime = L.realtime('http://127.0.0.1:5000/flightradar', {
    // interval of data refresh (in milliseconds)
    interval: 10 * 1000,
    getFeatureId: function(feature) {
        // required for L.Realtime to track which feature is which
        // over consecutive data requests.
        return feature.id;
    },
    pointToLayer: function(feature, latlng) {
        // style the aeroplane loction markers with L.DivIcons
        var marker = L.marker(latlng, {
            icon: L.divIcon({
                className:'aeroplane-visible',
                iconSize: [10,10]
            }),
            riseOnHover: true
        }).bindTooltip(
            // and as we're already here, bind a tooltip based on feature
            // property values
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
    }
}).addTo(map);
{{</ highlight >}}

We'll still need to define the looks of the aeroplane markers. As this map
will be in the style of a simple radar, then a let's have it a square with a
greenish color. Remember we added the markers in the `pointToLayer` function
with an extra className value of `aeroplane-visible` so simply define

{{< highlight css >}}
.aeroplane-visible {
    background: #109856;
    border: none;
    opacity: 1.0;
}
{{</ highlight >}}

in the `style` of the html file.

Don't forget to add the required attribution aswell (this goes in the `script`
tag of html `body`)

{{< highlight javascript >}}
map.attributionControl.addAttribution(
    '<br>Marker animation: <a href="https://github.com/perliedman/leaflet-realtime">Leaflet Realtime</a>'
);
map.attributionControl.addAttribution(
    '<br>Air traffic location data from <a href="http://www.opensky-network.org">The OpenSky Network</a>\'s public <a href="https://opensky-network.org/apidoc/">API</a>'
);
{{</ highlight >}}

Save the file and reload it in the web browser
and behold! There's a bunch of green squares moving around every n seconds that
pass.

{{< leaflet-realtime
    id="1"
    height="350px" width="100%"
    realtimeURL="https://tkardi.ee/current/flightradar/?format=json" >}}

All the mouseover events with tooltips opening and closing are done
automatically so this will give more time and space to play around with other
options. For example switch from using the divIcon-styled markers to
one of the aeroplanes from [Font Awesome](https://fontawesome.com/) (see
[Leaflet.awesome-markers](https://github.com/lvoogdt/Leaflet.awesome-markers)
plugin on how to do that), maybe scale the icons according to some feature's
properties and rotate them according to the direction travel.

But anyway, instead what I wanted was to have a greenish radar _hand_ moving
around with the aeroplane markers slowly fading out as the _hand_ has passed
over them something in the line of:

![https://giphy.com/gifs/radar-ki1rmMhjvLlm via GIPHY](
https://i.giphy.com/media/ki1rmMhjvLlm/giphy.webp)

For that we'll need to set up a polar graticule and a 360-degrees revolving
line, and find a way to animate the aeroplane markers in order to simulate the
look of a real radar. But I'll discuss that in the
[next writeup](../../30/building-a-totally-useless-map-ii/). The full
working code for `flightradar.html` as well as `proxy.py` and `flaskapp.py`
can be found in [this gist](
https://gist.github.com/tkardi/f9cc6199713adf14f8525ec4494b6710).

Happy exploring! :)
