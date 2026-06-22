---
title: "What a Data Lake Actually Is (and why you probably don’t need one)"
source: "https://thebuild.com/blog/2026/05/06/what-a-data-lake-actually-is-and-why-you-probably-dont-need-one/"
author: ""
published: ""
clipped: "2026-05-07T14:31:48"
tags:
  - clipping
  - data-lake
---

# What a Data Lake Actually Is (and why you probably don’t need one)

来源：[原文](https://thebuild.com/blog/2026/05/06/what-a-data-lake-actually-is-and-why-you-probably-dont-need-one/)

![](../../assets/data-lake-actually-is/data-lake-actually-is-01.png)

If your engineers have told you that you need a data lake, you should be a little suspicious. Most organizations that build data lakes don’t need them, and a substantial fraction of the ones that do build them end up with what the industry — without any irony — calls a “data swamp.” So before we get to what a data lake is, let me say plainly: the right answer is often “not yet, and maybe never.” The interesting question is when “yet” becomes “now.”

To get there, you have to understand that you actually have three distinct categories of data system, and they do different jobs.

**The transactional database** is what runs your application. PostgreSQL, MySQL, SQL Server. It is optimized for many small reads and writes — order placed, inventory decremented, user authenticated. Schema is strict: every column has a type, every row matches a table definition, every constraint is enforced. This is exactly what you want when correctness matters and the workload is predictable.

**The data warehouse** is where you do analytics. Redshift, Snowflake, BigQuery. The workload is the opposite: a smaller number of much larger queries that scan billions of rows and aggregate them. Warehouses are columnar, distributed, and tuned for the “how did sales go last quarter, broken down by region and product line” question that would make a transactional database miserable. Schema is still strict — you load the data into defined tables with defined columns.

**The data lake** is the third thing. It is a pile of files in cheap object storage — S3, GCS, Azure Blob — in whatever shape the data happens to arrive. CSVs from a vendor. JSON event streams from your application. Parquet files exported from your warehouse. Log files. Images. Maybe a directory of PDFs nobody has looked at in two years. The defining property is that the storage layer does not care about schema. You apply schema *when you read* , not when you write.

That last sentence is the whole point, so it is worth dwelling on. A warehouse asks you to commit to a schema before the data arrives. A lake does not. This is a bigger deal than it sounds. It means you can ingest data whose shape you don’t yet understand, from sources you don’t control, in volumes that would make a warehouse expensive — and you can decide later, possibly years later, what questions to ask of it.

(This is the concept on which Palantir, for better or worse, has built its entire business.)


### Why not just a bigger PostgreSQL?

PostgreSQL is a transactional database. It is very good at being one. It is not optimized for scanning a year of clickstream events to compute a funnel. You can absolutely put a few terabytes in PostgreSQL and run analytics on it, and for plenty of companies that is the right answer for a long time. But once you are past, say, ten terabytes of analytical data, or you are scanning hundreds of millions of rows per query, the row-oriented heap layout that makes PostgreSQL great at OLTP is working against you.

There are extensions and derivatives that change this: `citus` for distribution, `hydra` for column storage, ParadeDB for analytics-shaped PostgreSQL. They are great options. But at that point you have stopped using PostgreSQL-as-PostgreSQL and started using PostgreSQL as a substrate for something else, and the comparison to a purpose-built warehouse stops being lopsided in PostgreSQL’s favor.


### Why not just Redshift (or any other data warehouse)?

This one is more interesting, because Redshift is genuinely a warehouse and will handle the analytics workload. The reason it is not a complete answer is that warehouses force you to model the data on the way in. If you have a hundred sources producing wildly heterogeneous data, you spend an enormous amount of engineering effort writing ETL jobs to transform each source into the warehouse schema *before* the data is queryable.

Two consequences. First, anything you decided not to load is gone — or at least gone unless you go back to the source, which may no longer exist or may charge you for it. Second, every change to the schema requires reworking the ETL. The lake reverses this: you keep the original, untransformed data forever, and you transform on demand. If next quarter someone wants to ask a question your existing warehouse model cannot answer, the raw data is still there.


### When you actually need one

You need a data lake when one or more of the following is true:

- You have **genuinely heterogeneous sources** — logs, third-party feeds, scraped data, semi-structured events — that do not naturally fit a warehouse schema.
- You have **machine-learning or data-science workloads** that need raw, untransformed data, not the modeled subset your BI team built.
- You have **enough volume** that warehouse storage cost becomes a real line item rather than a rounding error.
- You expect the **questions to evolve faster than the schema can** , and you want to preserve optionality.

If none of those is true, you probably do not need a lake. You need a warehouse, possibly fed by PostgreSQL logical replication or a CDC pipeline, and you need to stop worrying about it.


### The lakehouse, briefly

The honest answer in 2026 is that the lake/warehouse distinction is collapsing. Open table formats — Apache Iceberg, Delta Lake, Apache Hudi — let you keep your data in object storage *and* have transactional tables, schema evolution, and time-travel queries on top of it. This is what the industry calls a “lakehouse.” Snowflake, Databricks, BigQuery, and Redshift all read Iceberg now, with varying degrees of enthusiasm. If you are building a new analytics stack today, the question is less “lake or warehouse” and more “which engine queries my Iceberg tables.”

This matters for your decision because it removes the worst part of the historical tradeoff. You no longer have to choose between “the lake, where everything goes but nothing has structure” and “the warehouse, where everything has structure but you had to know in advance.” You can have one storage layer that the lake people and the warehouse people both query.

What this means for you, practically: tell your engineers to bring you a one-pager that answers three questions. What specific decisions does this lake unblock that you cannot make today? What does it cost — in object storage, in compute, and in engineering time to operate? Who owns it on an ongoing basis, including the unglamorous parts like cataloging, access control, and deciding what to delete?

If they cannot answer all three, they do not need a lake yet. They need a meeting.
