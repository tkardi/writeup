---
title: "ESRI sde and db schemas"
date: 2023-07-28T15:37:51+02:00
draft: false
description: "On PostgreSQL DB schemas, usernames, ESRI attribute rules, and this oddity they call Arcade."
---

A colleague on the project I've been working on recently (which is unfortunately
built on ESRI technology stack) stumbled upon an oddity with an attribute rule
that is supposed to manage a recursive relationship between a parent row in
one table and child rows in another. This is essentially trying to model a
`primary key` - `foreign key` relationship with a `on delete cascade`
_delete action_ for the FK in the ESRI world.

And the implementation in Arcade (brrrrrr... why?) is pretty straight forward:

```javascript
// attribute rule to run ON DELETE for <parent-table>
// load related features
var related_features = FeatureSetByRelationshipName($feature, "<relationship-class-name>")

// filter related features
var gid = $feature.globalID;
var filtered_related_features = Filter(related_features, "<parent_id_column_name> = @gid")

// create and fill the deletes array
var deletes = []
for(var f in filtered_related_features) {
    Push(deletes, {"globalID": f.GlobalID})
}

// return
return {
    "edit": [
        {
            "className": "<child-parent-relates-table>",
            "deletes": deletes
        }
    ]
}
```

But the problem at hand: while it works in the dev database, it throws some
obscure error about the relationship class not found in the test database.
Check the database - it **IS** there...

Which conveniently brings us to the topic of how ESRI handles database schemas
and database usernames.

Short answer: it doesn't...

Well in documentation it does, but looking through the db sde schema functions
and doing some tests you can see that for some operations for whatever unknown
reasons, it uses them interchangeably. So the first thing to remember:

> If you want your data to be in schema xyz then you need to have a user role
> called xyz, who needs to be the physical owner of the schema. Granting all
> privileges
> for the schema xyz to user xyz is not enough although permissions are all
> there - this is something that esri
> checks during it's table-creation init routines.

So what, I hear you say.

True. But imagine you have hundreds of tables and you want to organize these
into logical stores in a database - this means that in addition to creating
n number of database schemas you will need to create and manage also a n
number of fictional database users whose only purpose of existence is to be a
line in

```sql
create schema if not exists foobar authorization foobar;
```

Which brings us to the subject of `search_path`s - the order in which schemas
that the database will be searched through for the objects you are referencing
in your queries, etc. Unless the schema for the function/table/whatnot is on
your active `search_path`, you'll need to schema-bind it. Meaning you'll have to
write:

```sql
select * from foo.bar where 1=0;
```

unless you have schema `foo` set on the `search_path`. In which case you could
discard it. But that might cause troubles otherwise.

And usually you would never have to worry about these because these are set up
to a rather reasonable default with SDE:

```sql
alter database <databasename> set search_path to "$user", public, sde;
```

Which means the connecting username (or schemaname) comes first, then `public` and
finally `sde`. Rather reasonable, except in some cases.

For example. And this case is also out of attribute rules and arcade. So it
turns out ESRI has finally made it possible to use an auto-incrementing (serial)
value for a column. This is achieved through creating a database sequence and
then creating an attribute rule in the lines of

```javascript
return NextSequenceValue("<sequence_name>");
```

for assignment to the column. Sweet, right?

But did you know that unless you are connecting as a schema owner of the
sequence+table in question you will get some horrible error message about the
sequence not being there? Well just schema-bind the damn thing then, meaning:

```javascript
return NextSequenceValue("<schema_name>.<sequence_name>");
```

and rejoyce on being able to edit the data without using the schema-owner
based connection. And **NB!** you'd still need the `INSERT`, `UPDATE`, `DELETE`
privileges to be granted to the whatever username you're connecting as.

But what is this? When you try to import that same thing to a gdb now the darn
XML workspace will not import anymore with some cryptic error message about
the sequence name being invalid...

![The sequence name is invalid](../img/xml-error.png)

So with all the cross-platform-compatibility talk, no it doesn't work like this.
The way we got it solved was by updating the `search_path` for the PostgreSQL
database to include the said schema:

```sql
alter database <databasename> set search_path to "$user", <schemaname>, public, sde;
```

But I think it's clear that in the long run this is very counter-productive:
when running on multiple schemas you are bound to end up with name clashes,
and incrementally growing the `search_path` to include yet another schema
seems a bit odd.

If the only reason you are dividing your datasets between schemas is user
permissions, then yeah, sure. But it's not the only way to manage it. A
convenient way, yes. But not the only one. So here's the second takeaway from
today:

> Don't bother using different schemas with ESRI SDE. It might look fancy
> and professional but in the end you'll end up with an administrative
> burden that none of the dev-ops/helpdesk people in your org will want to
> deal with.

Which gently brings me to the thing that I really wanted to write about in the
first place: ESRI relationship classes. As such they don't exist physically
in the database - they are simply rows of xml-blobs in the `sde.gdb_items` table.

```sql
select
    i.*
from
    sde.gdb_items i,
    sde.gdb_itemtypes t
where
    t.uuid = i.type and
    t.name = 'Relationship Class'
;
```
will list you all of them. As these are rows in a whatever-table then user
privileges don't apply. Meaning you see them in your flavour of ArcGIS desktop
application you're using at any rate either you have the permission for the
related tables or you don't. The name of the relationship class in a RDBMS like
PostgreSQL will be in the lines of

```
<databasename>.<schemaname>.<relationship-class-name>
```

sometimes confusing people like this is something real in the DB. No, it's not.
And because it's not a "_real thing_" it does not conform to the `search_path`
value that I was talking about. Here as long as you are the schema owner
`foobardata` you can reference your relationship class
`mydatabase.foobardata.this__that` between tables `foobardata.this` and
`foobardata.that` as `this__that`. Most probably because ESRI makes the
assumption that the relationshipclass name is prefixed by your dbname and
username you're using for connection.

As soon as you connect as another user you'll have to use the full path,
otherwise the thing will not be found:

![Failed to evaluate arcade expression](../img/attribute-rule-error.png)

Examining the XML workspace that ESRI produces, the Arcade expression in the
very beginning of this post for the `return`-clause `className` gets nicely
fixed from

```javascript
  [..]
// return
return {
    "edit": [
        {
            "className": "<dev-database>.<schemaname>.<child-parent-relates-table>",
            "deletes": deletes
        }
    ]
}
```

to

```javascript
  [..]
// return
return {
    "edit": [
        {
            "className": "<test-database>.<schemaname>.<child-parent-relates-table>",
            "deletes": deletes
        }
    ]
}
```

But the same thing is not done for the relationship class name that will remain
as

```javascript
// attribute rule to run ON DELETE for <parent-table>
// load related features
var related_features = FeatureSetByRelationshipName($feature, "<dev-database>.<schemaname>.<relationship-class-name>")
  [..]
```

And mind you again: don't try the dot notation for gdbs - it will throw up on
XML workspace import.

In conclusion we ended getting rid of schemanames for the arcade expressions and
now know that we'll need to:

1. update `search_path` for the database any time the database itself is built
   from scrtach
2. any time you do a "new deploy" of the data model go through the arcade
   expressions by hand to fix the relationship class names to point to the
   correct things.
