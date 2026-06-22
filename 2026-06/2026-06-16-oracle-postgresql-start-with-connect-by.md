---
title: 'Oracle to PostgreSQL: START WITH/CONNECT BY | EDB'
source: https://www.enterprisedb.com/blog/oracle-postgresql-start-withconnect-0
author: Kirk Roybal
published: June 08, 2020
saved: 2026-06-16 15:29 +0800
tags:
- postgres
- oracle
- connect-by
- recursive-cte
- migration
---

# Oracle to PostgreSQL: START WITH/CONNECT BY | EDB

Source: [https://www.enterprisedb.com/blog/oracle-postgresql-start-withconnect-0](https://www.enterprisedb.com/blog/oracle-postgresql-start-withconnect-0)  
Author: Kirk Roybal  
Published: June 08, 2020  
Saved: 2026-06-16 15:29 +0800

[Back to blog](/blog)

# Oracle to PostgreSQL: START WITH/CONNECT BY

![](/sites/default/files/styles/medium_scale_and_crop_220x220_/public/default_images/android-chrome-192x192.png?h=b7725292&itok=7O3Bc7t1)

##### [Kirk Roybal](/blog/author/kirk-roybal)

June 08, 2020

And now we arrive at the second article in our migration from Oracle to PostgreSQL series. This time we’ll be taking a look at the `START WITH/CONNECT BY` construct.

In Oracle, `START WITH/CONNECT BY` is used to create a singly linked list structure starting at a given sentinel row. The linked list may take the form of a tree, and has no balancing requirement.

To illustrate, let’s start with a query, and presume that the table has 5 rows in it.

```
SELECT * FROM person;
 last_name  | first_name | id | parent_id
------------+------------+----+-----------
 Dunstan    | Andrew     |  1 |    (null)
 Roybal     | Kirk       |  2 |         1
 Riggs      | Simon      |  3 |         1
 Eisentraut | Peter      |  4 |         1
 Thomas     | Shaun      |  5 |         3
(5 rows)
```

Here is the hierarchic query of the table using Oracle syntax.

```
select id, parent_id
from person
start with parent_id IS NULL
connect by prior id = parent_id;
 id | parent_id
----+-----------
  1 |    (null)
  4 |         1
  3 |         1
  2 |         1
  5 |         3
```

And here it is again using PostgreSQL.

```
WITH RECURSIVE a AS (
SELECT id, parent_id
FROM person
WHERE parent_id IS NULL
UNION ALL
SELECT d.id, d.parent_id
FROM person d
JOIN a ON a.id = d.parent_id )
SELECT id, parent_id FROM a;
 id | parent_id
----+-----------
  1 |    (null)
  4 |         1
  3 |         1
  2 |         1
  5 |         3
(5 rows)
```

This query makes use of a lot of PostgreSQL features, so let’s go through it slowly.

```
WITH RECURSIVE
```

This is a “Common Table Expression” (CTE). It defines a set of queries that will be executed in the same statement, not just in the same transaction. You may have any number of parenthetical expressions, and a final statement. For this usage, we only need one. By declaring that statement to be `RECURSIVE`, it will iteratively execute until no more rows are returned.

```
SELECT
UNION ALL
SELECT
```

This is a prescribed phrase for a recursive query. It is defined in the documentation as the method for distinguishing the starting point and recursion algorithm.  In Oracle terms, you can think of them as the START WITH clause unioned to the CONNECT BY clause.

```
JOIN a ON a.id = d.parent_id
```

This is a self-join to the CTE statement that provides the previous row data to the subsequent iteration.

To illustrate how this works, let’s add an iteration indicator to the query.

```
WITH RECURSIVE a AS (
SELECT id, parent_id, 1::integer recursion_level
FROM person
WHERE parent_id IS NULL
UNION ALL
SELECT d.id, d.parent_id, a.recursion_level +1
FROM person d
JOIN a ON a.id = d.parent_id )
SELECT * FROM a;

 id | parent_id | recursion_level
----+-----------+-----------------
  1 |    (null) |               1
  4 |         1 |               2
  3 |         1 |               2
  2 |         1 |               2
  5 |         3 |               3
(5 rows)
```

We initialize the recursion level indicator with a value. Note that in the rows that are returned, the first recursion level only occurs once. That’s because the first clause is only executed once.

The second clause is where the iterative magic happens. Here, we have visibility of the previous row data, along with the current row data. That allows us to perform the recursive calculations.

Simon Riggs has a very nice video about how to use this feature for graph database design. It’s highly informative, and you should take a look.

[![](/sites/default/files/blog_posts_images/2-14561-A_M-graphics-for-webinar-catalog-page-300x300.png.webp)](https://register.gotowebinar.com/register/8842412554362529805)

You might have noticed that this query could lead to a circular condition. That is correct. It is up to the developer to add a limiting clause to the second query to prevent this endless recursion. For example, only recursing 4 levels deep before just giving up.

```
WITH RECURSIVE a AS (
SELECT id, parent_id, 1::integer recursion_level  --<-- initialize it here
FROM person
WHERE parent_id IS NULL
UNION ALL
SELECT d.id, d.parent_id, a.recursion_level +1    --<-- iteration increment
FROM person d
JOIN a ON a.id = d.parent_id
WHERE d.recursion_level <= 4  --<-- bail out here
) SELECT * FROM a;
```

The column names and data types are determined by the first clause. Notice that the example uses a casting operator for the recursion level. In a very deep graph, this data type could also be defined as `1::bigint recursion_level`.

This graph is very easy to visualize with a small shell script and the graphviz utility.

```
#!/bin/bash -
#===============================================================================
#
#          FILE: pggraph
#
#         USAGE: ./pggraph
#
#   DESCRIPTION:
#
#       OPTIONS: ---
#  REQUIREMENTS: ---
#          BUGS: ---
#         NOTES: ---
#        AUTHOR: Kirk Roybal (), kirk.roybal@2ndquadrant.com
#  ORGANIZATION:
#       CREATED: 04/21/2020 14:09
#      REVISION:  ---
#===============================================================================

set -o nounset                              # Treat unset variables as an error

dbhost=localhost
dbport=5432
dbuser=$USER
dbname=$USER
ScriptVersion="1.0"
output=$(basename $0).dot

#===  FUNCTION  ================================================================
#         NAME:  usage
#  DESCRIPTION:  Display usage information.
#===============================================================================
function usage ()
{
cat <<- EOT

  Usage :  ${0##/*/} [options] [--]

  Options:
  -h|host     name Database Host Name default:localhost
  -n|name     name Database Name      default:$USER
  -o|output   file Output file        default:$output.dot
  -p|port   number TCP/IP port        default:5432
  -u|user     name User name          default:$USER
  -v|version    Display script version

EOT
}    # ----------  end of function usage  ----------

#-----------------------------------------------------------------------
#  Handle command line arguments
#-----------------------------------------------------------------------

while getopts ":dh:n:o:p:u:v" opt
do
  case $opt in

    d|debug    )  set -x ;;

    h|host     )  dbhost="$OPTARG" ;;

    n|name     )  dbname="$OPTARG" ;;

    o|output   )  output="$OPTARG" ;;

    p|port     )  dbport=$OPTARG ;;

    u|user     )  dbuser=$OPTARG ;;

    v|version  )  echo "$0 -- Version $ScriptVersion"; exit 0   ;;

    \? )  echo -e "\n  Option does not exist : $OPTARG\n"
          usage; exit 1   ;;

  esac    # --- end of case ---
done
shift $(($OPTIND-1))

[[ -f "$output" ]] && rm "$output"

tee "$output" <<eof< span="">
digraph g {
    node [shape=rectangle]
    rankdir=LR
EOF

psql -h $dbhost -U $dbuser -d $dbname -p $dbport -qtAf cte.sql |
    sed -e 's/^/node/' -e 's/.*(null)|/node/' -e 's/^/\t/' -e 's/|[[:digit:]]*$//' |
    sed -e 's/|/ -> node/' | tee -a "$output"

tee -a "$output" <<eof< span="">
}
EOF

dot -Tpng "$output" > "${output/dot/png}"

[[ -f "$output" ]] && rm "$output"

open "${output/dot/png}"</eof<></eof<>
```

This script requires this SQL statement in a file called cte.sql

```
WITH RECURSIVE a AS (
SELECT id, parent_id, 1::integer recursion_level
FROM person
WHERE parent_id IS NULL
UNION ALL
SELECT d.id, d.parent_id, a.recursion_level +1
FROM person d
JOIN a ON a.id = d.parent_id )
SELECT parent_id, id, recursion_level FROM a;
```

Then you invoke it like this:

```
chmod +x pggraph
./pggraph
```

And you will see the resulting graph.

![](https://www.2ndquadrant.com/en/blog/oracle-to-postgresql-start-with-connect-by/pggraph1.png "Five Nodes")

Five Nodes![](/sites/default/files/blog_posts_images/pggraph1-300x189.png.webp)

```
INSERT INTO person (id, parent_id) VALUES (6,2);
```

Run the utility again, and see the immediate changes to your directed graph:

![](https://www.2ndquadrant.com/en/blog/oracle-to-postgresql-start-with-connect-by/pggraph2.png "Six Nodes")![](/sites/default/files/blog_posts_images/pggraph2-300x189.png.webp)

Six Nodes

Now, that wasn’t so hard now, was it?

Share this

## More Blogs

### More Blogs

### [Buildfarm Query API](/blog/buildfarm-query-api)

A colleague asked me recently if there was an API for querying the PostgreSQL Buildfarm database. I told him there was not. I'm aware that a number of people have...

June 11, 2026

### [Just Clear a Day: What We Learned Running an AI Security Hackathon](/blog/just-clear-day-what-we-learned-running-ai-security-hackathon)

Security teams are professionally trained to be skeptical of new technology. When AI showed up, our first instinct was to treat it like everything else, a new attack surface to...

June 04, 2026

### [PGConf.dev 2026: Our team’s sessions, working groups, and key takeaways](/blog/pgconfdev-2026-our-teams-sessions-working-groups-and-key-takeaways)

Last week, we attended the annual PGConf.dev as a Gold-level sponsor. While most PostgreSQL conferences usually attract users and DBAs, this event draws a strong mix of contributors and community...

May 29, 2026
