---
title: "sde subtype hell"
date: 2023-10-20T08:18:30+03:00
draft: false
description: "'Just use the database default value for input missing data' is not a correct way of putting it with ESRI sde"
---

Getting the database structure in PostgreSQL is pretty straight-forward
and helps to do some really neat things like dynamic SQL parsing with type-casts
and so on. For example to get all columns of a specific table with the
column order, column names, their formatted datatype expressions, nullability,
and default values you can go like:

```sql
select
    ns.nspname as schema_name,
    c.relname as table_name,
    a.attnum as ord,
    a.attname as column_name,
    format_type(a.atttypid, a.atttypmod) as data_type,
    a.attnotnull as not_null,
    pg_get_expr(def.adbin, def.adrelid) as default_value
from
    pg_namespace ns,
    pg_class c,
    pg_attribute a
        left join pg_attrdef def on
            a.attrelid = def.adrelid and
            a.attnum = def.adnum
where
    c.relnamespace = ns.oid and
    c.oid = a.attrelid and
    a.attnum > 0 and
    a.attisdropped = false and
    c.oid = 'hexgrid'::regclass::oid
order by attnum
```
which will output something in the lines of

![SQL query results for basic datastructure details in PostgreSQL](../img/pg_attribute.png)

Cool, so when inserting data into a PostgreSQL backed ESRI sde database from
e.g. csv files we can use the database to tell us all of this?

No. Sorry, not everything. You can get a basic set of datatypes (`varchar`,
`int`, `smallint`, `timestamp`, `st_geometry`) and the _nullability_, but
not for example default values which are stored in sde system tables. This is
most probably due to suptypes which for a particular table all go into the
same "physical" table but might have different defaults assigned for a particular
column. So how do you go about this?

This can be achieved by parsing the xml definition from the corresponding row
in `sde.gdb_items` table. But first let's look at _domains_.

These are essentially codelists that you can assign to table columns (and
by saying "table" I mean both in ESRI parlance: _tables_ and _feature classes_,
as essentially these are the same) and consist of a code and a corresponding
label. The code value is stored in the table, label is shown to the user. If you
input (by hand) a wrong code value, your ESRI-based frontend will not know how
to label it. All's good for the database but people sucked into esri applications
are still complaining.

One way to get over this when inserting raw data from csv files is by checking
the domains available during insert. And maybe you don't have codes, but
the labels instead. So instead of doing

```sql
insert into foo.bar (
    objectid, some_domain_column
)
select
    sde.next_rowid64('foo', 'bar') as objectid,
    case
        when rows.some_domain_column =
            'The answer to life, the universe, and everything'
                then 42
        else -1
    end as some_domain_column
from (
    -- the following should be a table select but lets just pretent for a second
    values (
        'The answer to life, the universe, and everything'
    )
) as rows (some_domain_column)
;
```

and then later discovering that `42` is not defined in the domains. Or maybe the
codes change so quickly that it's hard labour coding all of this into case-when
statements. But we can prepare domains as a database view:

```sql
create or replace view public.v_sde_domains as
select
    i.name as domain_name, x.*
from
    sde.gdb_items i,
    sde.gdb_itemtypes t,
    xmltable(
        '/GPCodedValueDomain2/CodedValues/CodedValue'
        passing i.definition
        columns
            code text path 'Code' not null,
            name text path 'Name'
    ) x
where
    t.uuid = i.type and
    t.name = 'Coded Value Domain'
;
```

And we can then use this in the previous insert query as:

```sql
select
    some_domain_column.code
from (
    -- the following should be a table select but lets just pretent for a second
    values (
        'The answer to life, the universe, and everything'
    )
) as rows (some_domain_column)
    left join
        public.v_sde_domains some_domain_column on
            some_domain_column.domain_name = 'the_answer' and
            some_domain_column.name = rows.some_domain_column
;
```

So even if the domain codes change very quickly and undecidedly, we can rest
assured that we'll be getting correct codes for existing values, just as they
are in the database. We can even mark the mistakes this way. For example we
can discern between a _missing value in the input data_ and a _messed up value
in the input data_ with:

```sql
select
    case
        when rows.some_domain_column is not null
            then coalesce(
                some_domain_column.code,
                -1
            )
        else -2
    end as some_domain_column
from
   [..]
```

assuming you want _messed up values_ to be marked with `-1` and _missing values_
with `-2`. Although my personal preference for marking missing values has
always been `NULL`, but somehow for whatever reasons, personal or
data-management-provider based, I see that there are people who don't like
`NULL`s.

Now with domains it seems pretty straight forward, these are separate "objects"
floating somewhere out there in the database. It is important to keep track of
"domain owner" and "domain name" values as these might get messed up. So for
example if there's multiple schemas and you pass your datamodels via the
XMLWorkspace document (export from one DB, import to another), then you
should pay attention to in what order are you
exporting and importing your xmls, and that the domain definitions remain the
same in all xmls you've made of all schemas.

Although (at least in sde-exported xmls) there's a thing called
"domain owner", but it seems really a waste of textfile
bytes as the domains list contains *all* the domains from the db all the time.
At least I've not seen any effect nor point in that.

Btw, this here sounds like another reason why not to use schemas for sde
databases - it absolutely unnecessarily complicates DB admin / dev-ops
activities.

But anyway, subtypes. Using subtypes in this way is a bit more complicated, but
also possible. Unlike domains, subtypes are bound to the table definition, they
don't exist independently of their respective tables. So in order to get them
we'll have to query and parse the `definition` xml from the `sde.gdb_items`
for every table. And as I said before, when I say _table_, I mean _table_
and _feature class_ as apparently these have differing root xpath
expressions in the definition xml...

```sql
select
    r[2] as nsname, r[3] as relname,
    coalesce(fe.col_name, ta.col_name) as subtype_col_name,
    coalesce(fe.default_subtype, ta.default_subtype) as default_subtype,
    s.subtype_name, s.subtype_code,
    c.attname, nullif(c.domain_name, ''), c.default_value
from
    sde.gdb_items i,
    sde.gdb_itemtypes t
        left join lateral
            xmltable(
                '/DETableInfo'
                passing i.definition
                columns
                    default_subtype text path 'DefaultSubtypeCode',
                    col_name text path 'SubtypeFieldName',
                    subtypes xml path 'Subtypes'
            ) ta on true
        left join lateral
            xmltable(
                '/DEFeatureClassInfo'
                passing i.definition
                columns
                    default_subtype text path 'DefaultSubtypeCode',
                    col_name text path 'SubtypeFieldName',
                    subtypes xml path 'Subtypes'
            ) fe on true
        join lateral
            xmltable(
                '/Subtypes/Subtype'
                passing coalesce(fe.subtypes, ta.subtypes)
                columns
                    subtype_name text path 'SubtypeName',
                    subtype_code int path 'SubtypeCode',
                    fieldinfos xml path 'FieldInfos'
            ) s on true
        join lateral
            xmltable(
                '/FieldInfos/SubtypeFieldInfo'
                passing s.fieldinfos
                columns
                    attname text path 'FieldName',
                    domain_name text path 'DomainName',
                    default_value text path 'DefaultValue'
            ) c on true
        join lateral
            string_to_array(i.name,'.') r on true
where
    t.uuid = i.type and
    t.name in ('Table', 'Feature Class')
;
```

This view would list us all tables in the db that have a subtype,
particular domains assigned to columns and their default values for that
subtype. So we get multiple rows per table but one for every column
(in a subtype). It might be good for overview, but maybe it would be
easier to use in subsequent queries if we took advantage of [PostgreSQL's
JSON](https://www.postgresql.org/docs/15/functions-json.html) functionality.
Essentially aggregating all columns and subtypes into a JSON blob so we end
up with one row per table (or to be more precise: _subtype column_):

```sql
create or replace view public.v_sde_subtypes as
select
    nsname, relname, subtype_col_name, default_subtype,
    jsonb_object_agg(subtype_name, subtype_code) as subtype,
    jsonb_object_agg(subtype_name, col) as cols_per_subtype
from (
    select
        r[2] as nsname, r[3] as relname,
        coalesce(fe.col_name, ta.col_name) as subtype_col_name,
        coalesce(fe.default_subtype, ta.default_subtype) as default_subtype,
        s.subtype_name, s.subtype_code,
        jsonb_object_agg(
            c.attname,
            jsonb_build_object(
                'domain',
                nullif(c.domain_name, ''),
                'default',
                c.default_value
            )
        ) as col
    from
        sde.gdb_items i,
        sde.gdb_itemtypes t
            left join lateral
                xmltable(
                    '/DETableInfo'
                    passing i.definition
                    columns
                        default_subtype text path 'DefaultSubtypeCode',
                        col_name text path 'SubtypeFieldName',
                        subtypes xml path 'Subtypes'
                ) ta on true
            left join lateral
                xmltable(
                    '/DEFeatureClassInfo'
                    passing i.definition
                    columns
                        default_subtype text path 'DefaultSubtypeCode',
                        col_name text path 'SubtypeFieldName',
                        subtypes xml path 'Subtypes'
                ) fe on true
            join lateral
                xmltable(
                    '/Subtypes/Subtype'
                    passing coalesce(fe.subtypes, ta.subtypes)
                    columns
                        subtype_name text path 'SubtypeName',
                        subtype_code int path 'SubtypeCode',
                        fieldinfos xml path 'FieldInfos'
                ) s on true
            join lateral
                xmltable(
                    '/FieldInfos/SubtypeFieldInfo'
                    passing s.fieldinfos
                    columns
                        attname text path 'FieldName',
                        domain_name text path 'DomainName',
                        default_value text path 'DefaultValue'
                ) c on true
            join lateral
                string_to_array(i.name,'.') r on true
    where
        t.uuid = i.type and
        t.name in ('Table', 'Feature Class')
    group by
        1,2,3,4,5,6
) d
group by
    nsname, relname, subtype_col_name,
    default_subtype
;
```

We can use this in queries like the one we did before. Imagine the table
that we were inserting to has a subtype field assigned to, let's say
a column called `type` and that classifies if it's a `Question` or an
`Answer` (so a subtype field). Let's select a question again (see comments in
the SQL for some explanations):


```sql
select
    case
        when rows.type is not null
            /* get subtype code for value if it exists and is correct,
               othewise encode it as -1
            */
            then coalesce(
                (subtype.subtype->>rows.type)::int,
                -1
            )
        else
            /* ..or if there's none in the input, use subtype default,
               otherwise encode it as -2. Everything goes red in
               in arcgis.
            */
            coalesce(
                subtype.default_subtype::int,
                -2
            )
    end as type,
    case
        when rows.some_domain_column is not null
            /* there's an inbound value for column foo.bar.some_domain_column.
               using the "assigned domains" data we have from left joining
               v_sde_domains to v_sde_subtypes get the appropriate code
               value for the domain for this column in this subtype.
               -1 here means that
               there is no appropriate code in domain we should be using
               (messed up data)
            */
            then coalesce(
                some_domain_column.code::int,
                -1
            )
        else
            /* there was no input value for column. Use the default value
               assigned for this column in this subtype. -2 here means that
               there was no default value set for the column */
            coalesce(
                (
                    (
                        (
                            subtype.cols_per_subtype->>rows.type
                        )::jsonb->>'some_domain_column'
                    )::jsonb->>'default'
                )::int,
                -2
            )
    end as some_domain_column
from (
    -- the following should be a table select but lets just pretent for a second
    values (
        'Question',
        'The answer to life, the universe, and everything'
    )
) as rows (type, some_domain_column)
    left join
        public.v_sde_subtypes subtype on
            subtype.nsname = 'foo' and
            subtype.relname = 'bar'
    left join
        public.v_sde_domains some_domain_column on
            /* subtype.cols_per_subtype["<subtype-name>"]["<column-name>"]["domain"]
               gives the domain name: expectedly `the_answer`
               but we don't have to hard-code it here
            */
            some_domain_column.domain_name = (
                (subtype.cols_per_subtype->>rows.type)::jsonb->>'some_domain_column'
            )::jsonb->>'domain' and
            some_domain_column.name = rows.some_domain_column

;
```

This final select could be made even more elaborate with e.g. `NOT NULL` checking
for insert columns too - so if there's no default value and the column allows
`NULL`s there's no apparent reason why we should insert a value to that column.
The used _error code_ (-2) should actually go only to those columns where
it really is needed.

But why am I writing this? Personal reflection, most probably. And a few things
to remember:

- importing data using ESRI tools will guard against ill-managed domains and
subtypes but is woefully slow. And it doesn't allow for very much control on how
to handle errors or smallish changes, or recodings in a simple no-nonsense way
during transferral of data from one table to another table. At least for such
a simple minded user as myself.
- using `psql` to import data from csv files is blazing fast (compared to E),
and SQL (especially if you have PostGIS enabled in your PostgreSQL database)
gives you enormous power over what can be done or fixed automatically,
repeatedly, running the whole import process over and over again until you like
the results that you see.
- And here's a couple of extra things: You want to
preserve ESRI globalid values for table rows? Sure! You want to preserve the
ESRI editor tracking data? Sure! If for whatever reason you want to retain
the original ESRI objectid values? Of course you can. But only via the
_`psql` and csv_ approach, not ESRI itself. At least not straight out of the
box.
- Did you know that
[ogr2ogr CSV driver](https://gdal.org/drivers/vector/csv.html#layer-creation-options)
can be configured to write all kinds of geospatial data into csv files too? :)


(sorry, was only teasing with this last point...)
