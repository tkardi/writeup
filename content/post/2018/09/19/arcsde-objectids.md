---
title: "ArcSDE objectid values"
date: 2018-09-19T07:00:00+03:00
draft: false
---

Some distant years ago working on an ArcSDE database in PostgreSQL I needed
to manipulate SDE-managed tables straight in the database and not through ArcGIS
desktop or any of its toolsets at the same time making it possible
for desktop users do the same for the same datasets. And by manage I mean your usual
insert/update/delete commands. It's really straight-forward but if you need your
data to be usable through ArcGIS afterwards as well then you really need to keep
an eye on the `objectid` values of tables (or ... feature classes).  

After working through some possible solutions (for
example creating the table in the database and registering it with SDE) the
simplest and cleanest solution which would be easily migratable through
different environments (dev->test->..->production) ended up being db column
default values.

The whole process then consisted of creating your tables/feature classes with
ArcGIS desktop tools (e.g through recordset XML imports which were stored in
a version control system) and then running some extra SQL to set up things on
the database side. Making life easier for DB administrators.

Revisiting the past now because somehow I stumbled upon the, ummmmm....
_syntactical complexity_ for aquiring `objectid`s in SQLServer powered SDE
database as compared to PostgreSQL.

**NOTE**: I'm not saying this is the best (or even the only) way of achieving
the possibility of managing your data flow in a database in accordance with
SDE rules so the whole thing will not blow up and render your dataset unusable
for ArcGIS. Please be advised that there are dangers involved and use this
approach only at your own risk!

**NOTE 2**: I'm writing the following PostgreSQL queries by heart - they should
be correct to as much as I can remember. Still, as I don't have access to
PostgreSQL powered SDE database at the moment I can't test what I'm saying
and I do apologize for any errors and omissions. If there's anything amiss
please drop me a line.

If you're interested then the official primer can be found
[here](http://desktop.arcgis.com/en/arcmap/latest/manage-data/using-sql-with-gdbs/workflow-using-sql-with-existing-feature-classes.htm)

## sde i-tables
First off - the `i-tables` in a SDE database. Those basically act as
DB sequences but are realized rather as autocreated tables with two supporting
functions, one for getting `oid` values and one for returning unused `oid`
values. Not really sure why, but my guess is that this architectural choice as
opposed to using native DB based sequences has to do with the possibility to
copy-paste your dataset over different database brands (and versions) including
the  Esri's own file geodatabase format. Although, if I remember correctly -
this reason actually doesn't make sense because any time you copy-paste a
a table or feature class its oid _sequence_ is rewritten anyway as 1-based (
or some cases 0-based) and gaps filled - so it really does not act as an
**object identifier** as the name suggests.

So any time you create a table/feature class in ArcGIS desktop a corresponding
`i-table` with its _support functions_ is created in your RDBMS. SDE
differentiates between the myriad of them by adding an incremental number to the
name. For example: assuming your table (or feature class) is in a db schema is
called `foo`, you'd end up with a table `foo.i<some_number>`, and functions
`foo.i<some_number>_get_ids(*args)` and `foo.i<some_number>_return_ids(*args)`.
The `<some_number>` variable is derived from the sde system table called
`sde.sde_table_registry` and maps to the corresponding `registration_id` value
for that table. In order to find the `registration_id` value for a table called
`bar` you'd execute a query:

{{< highlight sql >}}
select registration_id
from sde.sde_table_registry
where table_name = 'bar';
{{</ highlight >}}

Now it gets tricky if we have tables (... or feature classes) with the
same name in different DB schemas. SDE has notion of RDBMS schemas but readily
confuses them with database users (or roles). This means that in order to be
able to create a database table `foo.bar` using ArcGIS you'd actually need to
use a sde connection with the username `foo`. Which in turn means that the
previous query taking into account also the db schema (or in reality - the
table's "owner") would be:

{{< highlight sql >}}
select registration_id
from sde.sde_table_registry
where table_name = 'bar' and schema = 'foo';
{{</ highlight >}}

Although then again, even this ownership of a relation is rather on a logical
not physical db level - the ownership is not reinforced in the RDBMS. It's
simply a column that sometimes is used as a DB username and sometimes a DB
schema. As it gets very messy and confusing and error-prone for ArcGIS very
fast so better stick with the idea that `DB schema` = `DB username` = most
probably `DB relation owner` because the schema should be owned by that user,
and don't ask any difficult questions. And... if this was not enough - if
you're working with SQLServer instead of PostgreSQL then
you have to use `owner` instead of `schema` in the previous query.

But coming back to the original subject. The same table (
`sde.sde_table_registry`) will also tell you the name of the
objectid column for a relation. Which usually is `objectid`. Except for those
cases when it isn't.

Anyway. Now that we have a `registration_id` for the table / feature class (
imagine it's `15` for example for our `foo.bar` table) we're interested in, we
can do a

{{< highlight sql >}}
select *
from foo.i15
{{</ highlight >}}

which returns something in the line of

| id_type | base_id | num_ids | last_id |
| ------- | ------- | ------- | ------- |
| 2       | 5342    | -1      | 5342    |

Most probably the only point to make here is the `base_id` column
which tells us that the last number that was pulled out of this pseudo-sequence
was 5342 and `id_type` which in the case of an _objectid-kinda-rowid-type_ seems
to be 2.

We're not going to go much more into detail with this as most probably
hand-managing this table will bring about apocalypse for the SDE itself and
other ArcGIS products depending on it. If you're interested then there's some
official documentation available for this on the inter-webs aswell.

Instead we'll use the aforementioned function to retrieve the row identifier
values.

### Getting fresh objectids
In order to reserve a fresh unused `objectid`, the simplest way is to ask for it
from the corresponding _i-getid-function_. It takes two parameters - the
id type and the number of ids you'd like to _reserve_.

Knowing the table's registration id (our `foo.bar` was 15 which we established
before) we can do (PostgreSQL-style)

{{< highlight sql >}}
select foo.i15_get_ids(2, 1).*;
{{</ highlight >}}

returning

| o_base_id | o_num_ids |
| --------- | --------- |
| 5343      | 1         |

so now we know that we have 1 objectid value (5543) that we can use without
confusing the SDE. If you need 10 fresh objectids then simply ask for 10:

{{< highlight sql >}}
select foo.i15_get_ids(2, 10).*;
{{</ highlight >}}

returning

| o_base_id | o_num_ids |
| --------- | --------- |
| 5344      | 10        |


### Returning unused objectids
Unless you're planning on having more rows in your table / feature class than
the max 32bit integer, then don't. And if you are still planning to, then you
could be better off with not using ArcSDE at all.

## Wrapping up
To achieve no more further hassle and get on with your inserts in the DB and
letting data be managed in desktop aswell, the simplest solution (for
PostgreSQL) to get _default next_ objectid is to wrap this up in a function
like

{{< highlight sql >}}
drop function if exists public.i_get_objectid(varchar, varchar);

create function public.i_get_objectid(table_schema varchar, table_name varchar)
returns integer as
$$
declare
    oid integer;
    rid integer;
    --safeguarding against possible injection
    _table_name regclass := format('%I.%I', table_schema, table_name);
begin
    execute '
        select registration_id as rid
        from sde.sde_table_registry
        where schema = $1
        and table_name = $2'
        into rid
        using table_schema, table_name;
    if rid is null then
        raise undefined_table
            using message = format(
                'Relation "%s.%s" is not registered with SDE!',
                table_schema, table_name
            );
    end if;
    execute
        format(
            'select o_base_id from %s.i%s_get_ids(2, 1)',
            table_schema, rid
        )
    into oid;
    return oid:
end;
$$
languge plpgsql security invoker;
comment on function public.i_get_objectid(varchar, varchar) is
    'Get new ESRI objectid for table in schema. Will not check for invoker''s '
    'INSERT privilege for relation but does check for input relation''s existence '
    'and registration status with SDE.';
{{</ highlight >}}

and then, granted that all the execution privileges have been set accordingly

we can simply

{{< highlight sql >}}
alter table foo.bar
    alter column objectid set default public.i_get_objectid('foo', 'bar');
{{</ highlight >}}

to reduce the hassle during in-database inserts to this table.

## Why?
This _why_ is not actually about why this db-default-value driven approach
should be used. Unless there's no other possibility - don't use it :). And
with the availability (and usability) of open source GIS software today there's
always one other option - stop using ArcSDE altogether.

This _why_ is more about the SQLServer thing I was talking about before. Remember,
in PG we could simply

{{< highlight sql >}}
select foo.i15_get_ids(2, 10).*;
{{</ highlight >}}

Right? In SQLServer this same result is achieved by

{{< highlight sql >}}
DECLARE @o_baseid int, @o_numobtained int;
EXEC foo.i15_get_ids
    @id_type = 2, @num_requested_ids = 10,
    @base_id = @o_baseid OUTPUT, @num_obtained_ids = @o_numobtained OUTPUT;
SELECT @o_baseid as baseid, @o_numobtained as numobtained;
{{</ highlight >}}

So, yes...
