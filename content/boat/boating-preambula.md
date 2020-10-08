---
title: "Building a boat again"
date: 2020-09-07T08:08:58+02:00
draft: false
---

So... it could have been the 2020 quarantine blues - or as a friend later
suggested - the onset of mid-life crises. Probably both or something whatever
completely else although I must admit the only things relating to mid-life
crises for me up to now is the song by
[_Faith No More_ ](https://www.youtube.com/watch?v=K8n_lKuIpK0)
and an Estonian 1986 movie _Keskea rõõmud_ (
[_The Joys of Midlife_](https://www.efis.ee/en/film-categotries/movies/id/878)) -
with the term used in the movie title oddly in colloquial language also used
to denote _longjohns_.

Anyway. Somewhere in the beginning of May the idea formed that a cruising
sailboat is required and as with the previous boat I'll be trying to DIY with
maybe only some parts purchased or done somewhere by somebody else. As with the
previous boat I found the plans I like off
[Svenson's free boat plans](http://www.svensons.com/boat/)
page which hosts a myriad of plans from Science and Mechanics and
Boat Builder Handbooks of the old. The one that I liked and decided to go for
is this [24ft motor-sailer](http://www.svensons.com/boat/?p=SailBoats/Gypsy)
by William D. Jackson.

All's nice and all but since this will be a project spanning multiple years then
(drawing on previous experience) it would be better to have some kind of roof
over and walls around the building space. And even if opting for building
outside there's going to be the problem of finding a suitable _lofting_ space
for drawing up full sized plans of the boat. Because nails will be driven into
the floor for drawing faired arcs.

# Building space
Speaking in geographic terms, the area of the boat's _bounding box_ would be
approximately 2.75m * 7.5m = 20.6m2. And this discards all requirements of
needing space around the boat skeleton during build aswell.

Now, the problem is that the government regulations require all buildings with
(some legal this-and-thats but as a rule of thumb) an orthogonal projection area
more than 20m2 to have proper plans drawn up, a ton of permissions acquired,
state fees paid etc.

Powered with a pencil and a piece of paper I set out to shape the shed so that
it would have some space around the boat and at the same time minimize the
on-the-ground area. The final design that I came up with is actually shaped
like a boat and expressed as a PostGIS query as

{{< highlight sql >}}
select
    st_buildarea(
		st_rotate(st_translate(st_makeline(array[
		st_point(0, 1.5),
		st_point(1.5, 3.0),
		st_point(5.05, 3.0),
		st_point(7.5, 2.75),
		st_point(7.5, 1.5),
		st_point(7.5, 0.25),
		st_point(5.05, 0.0),
		st_point(1.5, 0.0),
		st_point(0, 1.5)
	]), pnt.x, pnt.y), pi()*0.80, pnt.x, pnt.y)) as shed
from (select 0 as x, 0 as y) as pnt;
/*which yields an onground area of just under 20m2*/
{{</ highlight >}}

In order to maximize building space all walls have been designed as _lift-up doors_.
Although this in turn means that since diagonal supports will not extend the
full sides of walls the construction will be a wee weaker. To mitigate that I'm
keeping temporary diagonal supports at locations so they can be removed and
placed in a different location as required. The shed's _stern_ (pun intended
here and afterwards) is 2.5m wide and moving forward this increases to 3m at
about the boat's station #6 as seen in the following photo.

![Boatshed's aft view with 3 lift-up walls/doors on the starboard :) side
and one in the stern. There are similarly 3 lift-up walls on the port side
](../img/shed-aft.jpg)

In the fore it has a triangularly shaped roof extending out 1.5m from the solid
floor covered area. This is the area where the stem part of the boat will be
built. The doors for the shed are quite light having only mosquito-netting as a
cover (because of missing real diagonal supports). Hopefully this will keep
most of the rain/snow trying to come in sideways out. But will see by springtime
next year.

![Boatshed's fore view with the extended roof to cover the bow part of
the boat](../img/shed-fore.jpg)

I'm still hesitant if I'll be able to do the whole building process (including
planking the hull and building the deck and so on) inside. Could happen that I
will have to move outside already for attaching chines (two beams of appr
7.5m of 4.5*9.5cm) to the sides of frames connecting the aft
and fore ends of the boat. But there are a few options that might be considered
here.. but all in good time and will try to solve problems as they arise and
not worry too much what, when and how.

![Boatshed's inside looking fore to the outside](../img/shed-inside.jpg)

Still have a few things to finish with the shed but otherwise lofting can
begin.
