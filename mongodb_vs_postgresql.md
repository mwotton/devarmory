layout: page
title: MongoDB vs PostgreSQL: the smackdownening.
---

# How to get JSON search without installing MongoDB

If you've been a dev on the net in, say, the last decade, you've
probably seen an ad for MongoDB. The marketing was fierce - NoSQL was
coming, the proponents told us: no longer would we ever have to go to
the artistic-fervour-killing hassle of actually thinking about what
our data looked like and designing a database schema. Just throw
JSON-shaped bags at a persistent store, write your Node app and head
to the pub, chuckling at those poor enterprise drones labouring away
on joins and conditions and group-bys.

I'm not going to go into the details of why this was an incredibly bad idea.
[Sarah Mei](http://www.sarahmei.com/blog/2013/11/11/why-you-should-never-use-mongodb/)
has already done the hard yards there, in forensic, painful and
occasionally eye-wateringly graphic detail. I want to talk about the
odd occasions in which flippy-floppy JSON-based search is actually a better
idea than modeling every painful detail in tables, and how you don't
have to leave the comfortingly stable embrace of PostgreSQL in order
to get it.

Let's assume you want to offer some kind of search. If you have
control of both sides of it, then it's easy - you can dictate an
ontology of tags and other structured metadata in precise enough terms
that you can model it directly in tables. But what if you're expecting
to have to serve a multitude of different uses, such that the clients
expect to be able to throw in arbitrary metadata and also search by
it? If you try to model that in the database, you are setting yourself
up for endless rework & never-ending database migrations as you try to
cram new sources of data into it. That doesn't sound particularly
calm-inducing: must we therefore abandon SQL?

Thankfully, the answer is no: the PostgreSQL devs have in their
infinite wisdom included a fast and robust JSON search facility.

## Examples

Let's take some slightly cut-down code from the MongoDB quickstart:

```
db.inventory.insertMany([
   // MongoDB adds the _id field with an ObjectId if _id is not present
   { item: "journal", qty: 25, status: "A",
       size: { h: 14, w: 21, uom: "cm" }, tags: [ "blank", "red" ] },
   { item: "notebook", qty: 50, status: "D",
       size: { h: 8.5, w: 11, uom: "in" }, tags: [ "red" ] }
]);
```

and translate it into PostgreSQL. For the sake of simplicity, I'm
going to assume that the relevant data you actually want to pull out
is associated with the id of the `items` table elswhere in the database.

```
create table items (id serial, metadata jsonb);
insert into items ("metadata") values ('{ "item": "journal", "qty": 25, "status": "D",
       "size": { "h": 14, "w": 21, "uom": "cm" }, "tags": [ "blank", "red" ] }');
insert into items ("metadata") values ('{ "item": "notebook", "qty": 50, "status": "A",
       "size": { "h": 8.5, "w": 11, "uom": "in" }, "tags": [ "red" ] }');
```

We use `jsonb` as a column type rather than `json` because there are
more operators defined for it and it's faster.

(A reasonable person may complain at this point that this data is perfectly
regular and could be modeled relationally. This is true, but you can
presumably use your imagination to expand to a case where this is not
so. Also, pedantry doesn't get you invited to any of the cool
parties.)

So, how do we search it? Let's look at the first non-trivial mongo
query

```
db.inventory.find( { status: "D" } )
```

This translates as

```
select id from items where metadata->>'status'='A';
```

That was simple! What about something where you want to match a whole
object, like

```
db.inventory.find( { size: { h: 14, w: 21, uom: "cm" } } )
```
?

This translates as
```
select id from items where metadata @> '{ "size": { "h": 14, "w": 21, "uom": "cm" } }';
```

the `@>` operator lets us compute on JSON subsets.

What about array inclusion?

```
db.inventory.find( { tags: "blank" } )
```

(incidentally, the Mongo query here is ambiguous - there's no way to
indicate that you want to find the fragment `{tags : "blank"}` vs
looking for an array like `{tags: ["blank", "chartreuse", "mauve"]}`.)

In PostgreSQL:

```
select id from items where metadata->'tags' ? 'blank';
```

(The `?` operator works on JSON objects as well as arrays.)


What about if we wanted to match any one of a number of tags?

```
select id from items where metadata->'tags' ?| array['blank','red'];
```

or

```
select id from items where metadata->'tags' ?& array['blank','red'];
```

will give us everything matching both.

MongoDB also has some syntax like `$or` and `$and` to combine queries
(which I believe also means that if your data contains those as
literal keys, you are out of luck). PostgreSQL simply embeds the
JSON/JSONB operators into SQL, so you can use `AND` and `OR` as you
would with any other query.

## Go forth and sin relationally no more

JSON search is great sometimes. You shouldn't need to run two
databases or abandon relational modeling to get it though, and thanks
to PostgreSQL, you don't have to.
