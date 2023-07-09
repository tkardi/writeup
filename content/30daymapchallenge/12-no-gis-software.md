---
title: "2020 / Day 12: no gis software"
date: 2020-11-12T14:39:51+03:00
draft: false
og_image: "../img/30daymapchallenge/thumb.png"
---
### Data
- [OpenStreetMap contributors](https://www.openstreetmap.org/)

### Tools
PostGIS, GeoServer, shell.

### Summary
Back in July 2020 FOSS4G Europe was supposed to take place in Valmiera, Latvia.
But it didn't. Because of you know what. So I suddenly got this idea, that I need
to do a map of Valmiera as shout-out to my fellow LOCers. And remembering
[mapscii](https://github.com/rastapasta/mapscii) decided to do it in the terminal
with a simplistic bash script using GeoServer UTFGrid output from PostGIS
database.

```
#!/bin/bash
fg_black="$(tput setaf 0)"
fg_red="$(tput setaf 1)"
fg_green="$(tput setaf 2)"
fg_yellow="$(tput setaf 3)"
fg_blue="$(tput setaf 4)"
fg_magenta="$(tput setaf 5)"
fg_cyan="$(tput setaf 6)"
fg_white="$(tput setaf 7)"
reset="$(tput sgr0)"

data=$(curl "http://localhost:8080/geoserver/f/wms?service=WMS&version=1.1.0&request=GetMap&layers=f:admin,f:waters,f:builtup,f:roads&WIDTH=830&HEIGHT=250&BBOX=577579.7139896681%2C6373247.321590611%2C594758.7926965223%2C6381836.860944037&srs=EPSG%3A3301&format=application%2Fjson%3Btype%3Dutfgrid" | jq -r '.grid | join("\n")')

sleep 4

for (( i=0; i<${#data}; i++ )); do
 v=${data:$i:1}
 c=${fg_white}
 if [[ ${v} == ' ' ]]
 then
   v="¤"
 fi
 case $v in
  "!") c=${fg_yellow};;
  "#") c=${fg_yellow};;
  "&") c=${fg_yellow};;
  "%") c=${fg_yellow};;
  "(") c=${fg_black};;
  "'") c=${fg_red};;
  "$") c=${fg_blue};;
  "¤") c=${fg_green};;
 esac
 sleep 0.00001 | echo -ne "${c}${v}${reset}"
done
echo ""
```

Note the use of `sleep` for inbetween pause before echoing a char back on screen.
And I know I'm evil, but using epsg:3301 for Northern Latvia is ok. I guess... :D

[hi-res](https://tkardi.ee/writeup/img/30daymapchallenge/day-12-no-gis-software.gif)

{{< tweet user="tkardi" id="1326755068796936193" >}}
