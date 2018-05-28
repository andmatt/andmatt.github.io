---
title: "How I Learned to Stop Worrying and Love JSON"
date: 2018-05-5
excerpt: "A guide to using PostgreSQL in Python and storing complex metadata using JSON"
---
Oftentimes, I find myself in situations where I need to store complex nested metadata in a database. This can be a daunting task, as this first thing that comes to mind is creating a host of new metadata tables that each map a distinct element onto the primary key of your main table. However, this level of complexity can often be condensed into a single column by using the `JSON` or `JSONB` datatypes.   

__Note:__ This tutorial contains code specific to Postgres and Python, however these concepts can be applied to a variety of database frameworks.

## JSON and JSONB
`JSON` stands for JavaScript Object Notation, and is a human readable text based object derived from Javascript. A JSON file looks like this.
```python
{"First Name": "Matt",
"Last Name": "Guan",
"Age": "25"}
```
This is pretty much the same thing as a Python dictionary. There is a `name` (or `key`) on the outer level, with a `value` that can be accessed inside.

`JSONB` or JSON Blob is a unique Postgres datatype that is almost identical to JSON. The only real difference is that JSONB is stored in a decomposed binary format vs. text, which processes faster. Postgres also has a lot of built in stored procedures that work specifically with JSONB.

JSON and JSONB both have unique functions and operators, that allow for a lot of flexibility in both updating and calling data. These formats are supported with Postgres version 9.3 and above.

## A quick note on Using Postgres in Python
Throughout this tutorial, I will be writing a lot of queries in pure SQL. These can be wrapped as a multi-line string in Python and be executed by arguments within either Pandas or your database cursor.  

For executing postgres queries, I usually just use the default `psycogs2` cursor object. This can also be done just as easily by initializing a connection from a `sqlalchemy` engine, although the parameter substitution syntax will be a little different.  

```python
import psycopg2

#Can also use a sqlalchemy engine
conn = psycopg2.connect("dbname=test, user=postgres")
cur = conn.cursor()

s='''
INSERT INTO important_data (id, data)
VALUES (%(id)s, %(data)s);
'''

cur.execute(s, {'id':1, 'data':'Trade Secrets'})
```

For reading SQL in Python, it is actually easier to use the `pandas` `read_sql_query` argument and pass in the database connection. (The psycopg2 cursor will return output in a harder to use nested tuple format)

```python
import pandas as pd

s='''
SELECT * 
FROM important_data
WHERE id=%(id)s
'''

pd.read_sql_query(s, conn, params={'id':1})
```

<div class="notice--warning">
  <strong>Note:</strong> Always use parameter substitution rather than string parsing for SQL queries in Python! You will get weird data type mismatch errors otherwise. The only exception to this would be for table names.
</div>

## Creating a JSON Field
Creating a table with a JSON field is relatively simple - just use the built in JSON or JSONB datatype. The one you choose doesn't _really_ matter. JSONB is the preferred choice, but if you already created a table with a JSON column, there is no need to recreate it as Postgres has a built in stored procedure to convert JSON to JSONB (more on that later).

```sql
CREATE TABLE user_metadata
(userId INT PRIMARY KEY,
metadata JSONB,
publish_time TIMESTAMP DEFAULT current_timestamp);
```

## Generating Valid JSON in Python
Creating your JSON object to insert into Postgres is pretty easy. Just create a Python dictionary of any level of complexity, and then use the `dumps` argument from the `json` module to convert it to a string of valid JSON.  

Below is an example of creating a JSON file from a nested dictionary. I imagine that this is a small subset of what actually exists in some database at Google.  

Note that we can condense all of this data into one column! In a typical relational database, this could potentially span up to 5+ tables across various schemas.

```python
import json

metadata_dict = {
    'name': {'first': 'Matt',
             'last': 'Guan'},
    'demo': {'ethnicity': 'asian',
             'age': '25',
             'education': ['high school', 'bachelors']},
    'interests': {'lifestyle': ['fitness', 'healthy living', 'travel', 'slick deals'],
                  'hobbies': ['board games', 'live music', 'movies', 'anime betrayals']}
}

json_str = json.dumps(metadata_dict)
```

## Updating Your JSON Column
So now we've generated created our table, and populated our JSON column. So how do we easily update it? Luckily there is a lot of built in functionality for this.  

The easiest option is just to pull the entire JSON column into Python using `pandas`, update the dictionary, update the entire JSON, and load it back in. This is super easy, but can get cumbersome for more complex operations.  

Below is an example of inserting an entire JSON object back into the `user_metadata` table using the primary key of `user_id` along with `publish_time` to ensure that I am only overwriting one entry.

```sql
UPDATE user_metadata
SET metadata = %(insert_json)s::json
WHERE userId = %(user_id)
AND
publish_time = %(time)s
```

We can also update an element nested within a JSONB using `jsonb_set` - a built in Postgres function with the following parameters - `jsonb_set(target, path, new_value, create_missing)`
<p style="font: 19px 'Monaco';">
<strong>target, jsonb:</strong> the <code>JSONB</code> that will be modified <br>
<strong>path, text:</strong> Indicates the location of the field to update <br>
<strong>new_value, jsonb:</strong> the new <code>JSONB</code> that you want to insert <br>
<strong>create_missing, bool:</strong> If True, creates the path within the target <code>JSONB</code> if it did not exist before
</p>
This function only works on `JSONB` columns though! But like I said earlier, it's fine if your column is in `JSON` - you can just use another build in stored procedure `to_jsonb` to temporarily convert your column.  

This method can be useful for modifying multiple elements at once that reside deep within the JSONB, that would otherwise be a pain to parse out.  

Below is the code to update only the `interests` dictionary. Note that in this example, I am using the `to_jsonb` stored procedure and then converting the column back to JSON using `::json`. You would ignore these if your column was already JSONB.

```sql
UPDATE user_metadata
SET metadata = jsonb_set(to_jsonb(metadata), '{interests}', %(insert_dict)s, false)::json
WHERE userId = %(user_id)
AND
publish_time = %(time)s
```

We can even take this a step further and just modify just the array within the nested dictionary by extending the `target` argument.

```sql
UPDATE user_metadata
SET metadata = jsonb_set(to_jsonb(metadata), '{interests, hobbies}', %(insert_array)s, false)::json
WHERE userId = %(user_id)
AND
publish_time = %(time)s
```

## Querying JSON 
Lastly, there is build in functionality to query nested JSON data directly! Namely, `->` in addition to a `key` within the JSON will return the nested `value`. `->>` does the exact same thing, but returns the value as text. I usually just use this for the final call if I'm trying to get to a deeply nested element.  

Below is a query I would use to return the entire nested dictionary under the `demo` key. 

```sql
SELECT metadata->>'demo' as demographics
FROM user_metadata
WHERE user_id=%(user_id)s
```

If I want to take it a step further and just extract the age component, I would simply add a `->`

```sql
SELECT metadata->'demo'->>'age' as age
FROM user_metadata
WHERE user_id=%(user_id)s
```

Lastly, we can filter an entire table on a nested field by using the `->` notation with the `WHERE` statement. Just take a second to think about how useful this is. If you have an extremely robust metadata column, you can condense a wall of SQL code full of joins and subqueries into just 3 lines of code!

```sql
SELECT *
FROM user_metadata
WHERE metadata->'demo'->>'ethnicity' IN ('asian', 'white')
```

That should be enough to get you started on building out tables in Postgres with JSON. I hope this was helpful. Thanks for reading!
