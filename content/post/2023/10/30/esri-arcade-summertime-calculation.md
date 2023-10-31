---
title: "Timezone handling in esri ..."
date: 2023-10-30T21:44:32+02:00
draft: false
description: "Just because there's no better way to describe this situation: timezone handling in esri sucks."
---

... is horrible.

In a PostgreSQL backed SDE database all timestamp-type columns (dubbed as
`Date`) are stored as `timestamp without time zone` which essentially is not
a bad thing in itself. You can assume UTC everywhere and just go with the flow.
But sometimes there's a need of storing timestamps in different timezone
data in the same database or in the same table even. How do you go about it?
As the database column is `without time zone` it really doesn't matter
what you put in there as long as you remember (and your co-workers remember)
what was that timezone and how to calculate it. Meaning how many hours to add
or subtract.

To make things worse there is this thing called
[**_daylight saving time_**](https://en.wikipedia.org/wiki/Daylight_saving_time) (or
_summertime_), the end of which we celebrated here in Europe just this weekend.
By the way, it has taken me 40+ years to finally understand how this works:
you move the clock hands always **towards the northern hemisphere summer**,
like this:

```
Last Sunday of March --> all summer months <-- Last Sunday of October
```

So in the spring forward, in the autumn backward.

The more difficult thing is to determine _when_ the change occurs. For
the last-Sunday-of-October it seems to be a constant: it's the 43rd week (?),
at least as far back as I looked. But the spring date seems to jump
between the Sunday of week 12 and week 13.

But leaving that aside for the moment. Looking through the [**ESRI Arcade
documentation**](https://developers.arcgis.com/arcade/function-reference/date_functions/)
switching timezones seems like a really trivial thing. This is wrong. Most
of the _date_-functions are related to the latest release ([**1.24**](https://developers.arcgis.com/arcade/guide/version-matrix/)) which
seems to be packaged into ArcPro 3.2 which in turn does not seem to have a
[**release date**](https://support.esri.com/en-us/products/arcgis-pro/life-cycle)
scheduled at all yet? Tough luck.

This is exactly what I needed to do: based on a UTC timestamp column
that a user fills, calculate on the fly another timestamp column in the local
timezone.

The only thing we can do is blindly add an amount of time to a timestamp
with [**DateAdd**](https://developers.arcgis.com/arcade/function-reference/date_functions/#dateadd)?
And since Estonia is UTC +2h in winter and UTC +3h in the summer, you choose
one of these and get about half of your timestamps correct. And this is stupid.
But determined that there needs to be a way to do this _by hand_ aswell I
stumbled upon [**this post in Medium**](https://medium.com/capgemini-microsoft-team/determining-whether-a-date-is-within-daylight-savings-logic-app-flow-ff7119818ad4) which describes the summer-/wintertime date conversion
in TypeScript. Porting that to Arcade requires some more `if`s and I needed to
bring in hour-handling aswell. So 00:00 UTC in the spring and 01:00 UTC in autumn
is the _change point_. After some trial and error came up with this
monstrocity:

```js
var month = ISOMonth($feature.time_in_utc);

//January, February, November, and December are +2h.
if (month < 3 || month > 10) {
    return DateAdd(Date($feature.time_in_utc), 2, 'hours');
}

//April to September are +3h
if (month > 3 && month < 10) {
    return DateAdd(Date($feature.time_in_utc), 3, 'hours');
}

// Get DOW and convert it to js notation [Sunday - 0, Saturday - 6]
var dow = ISOWeekDay($feature.time_in_utc);
if (dow == 7) {
    dow = 0;
}
var previousSunday = Day($feature.time_in_utc) - dow;
var hh = Hour($feature.time_in_utc);

// In March,
if (month == 3) {
    // if previous Sunday was before the 25th.
    if (previousSunday < 25) {
        // We're in wintertime
        return DateAdd(Date($feature.time_in_utc), 2, 'hours');  
    }
    // Otherwise we're in summertime
    return DateAdd(Date($feature.time_in_utc), 3, 'hours');
}

// In October,
if (month == 10) {
    // if previous Sunday was before the 25th.
    if (previousSunday < 25) {
        // We're in summertime
        return DateAdd(Date($feature.time_in_utc), 3, 'hours');
    }
    // if previous Sunday was on or after the 25th
    // hh24 in question is less than 1 AM and it is a Sunday
    if (previousSunday >= 25 && hh < 1 && dow == 0) {
        // We're in summertime
        return DateAdd(Date($feature.time_in_utc), 3, 'hours');
    }
    // Otherwise we're in wintertime
    return DateAdd(Date($feature.time_in_utc), 2, 'hours');
}
```

And by the way, in PostgreSQL you'd do:

```sql
select
    '2023-10-29T01:00:00'::timestamp at time zone 'utc' at time zone 'Europe/Tallinn'
```

Conclusion?

Free-and-open-source rocks! There's so much with
"batteries included", and better organized documentation, and the possibility
to step up and fix things if anything is broken.
