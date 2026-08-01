---
title: "Chapter 59. Writing a Custom Scan Provider"
source: "https://www.postgresql.org/docs/17/custom-scan.html"
author:
published: 2026-05-14
created: 2026-07-29
description: "Chapter 59. Writing a Custom Scan Provider Table of Contents 59.1. Creating Custom Scan Paths 59.1.1. Custom Scan Path Callbacks 59.2. Creating …"
tags:
  - "clippings"
---
July 16, 2026: [PostgreSQL 19 Beta 2 Released!](https://www.postgresql.org/about/news/postgresql-19-beta-2-released-3350/)

Supported Versions: [Current](https://www.postgresql.org/docs/current/custom-scan.html "PostgreSQL 18 - Chapter 59. Writing a Custom Scan Provider") ([18](https://www.postgresql.org/docs/18/custom-scan.html "PostgreSQL 18 - Chapter 59. Writing a Custom Scan Provider")) / [17](https://www.postgresql.org/docs/17/custom-scan.html "PostgreSQL 17 - Chapter 59. Writing a Custom Scan Provider") / [16](https://www.postgresql.org/docs/16/custom-scan.html "PostgreSQL 16 - Chapter 59. Writing a Custom Scan Provider") / [15](https://www.postgresql.org/docs/15/custom-scan.html "PostgreSQL 15 - Chapter 59. Writing a Custom Scan Provider") / [14](https://www.postgresql.org/docs/14/custom-scan.html "PostgreSQL 14 - Chapter 59. Writing a Custom Scan Provider")

Development Versions: [19](https://www.postgresql.org/docs/19/custom-scan.html "PostgreSQL 19 - Chapter 59. Writing a Custom Scan Provider") / [devel](https://www.postgresql.org/docs/devel/custom-scan.html "PostgreSQL devel - Chapter 59. Writing a Custom Scan Provider")

Unsupported versions: [13](https://www.postgresql.org/docs/13/custom-scan.html "PostgreSQL 13 - Chapter 59. Writing a Custom Scan Provider") / [12](https://www.postgresql.org/docs/12/custom-scan.html "PostgreSQL 12 - Chapter 59. Writing a Custom Scan Provider") / [11](https://www.postgresql.org/docs/11/custom-scan.html "PostgreSQL 11 - Chapter 59. Writing a Custom Scan Provider") / [10](https://www.postgresql.org/docs/10/custom-scan.html "PostgreSQL 10 - Chapter 59. Writing a Custom Scan Provider") / [9.6](https://www.postgresql.org/docs/9.6/custom-scan.html "PostgreSQL 9.6 - Chapter 59. Writing a Custom Scan Provider") / [9.5](https://www.postgresql.org/docs/9.5/custom-scan.html "PostgreSQL 9.5 - Chapter 59. Writing a Custom Scan Provider")

<table width="100%"><tbody><tr><th colspan="5" align="center">Chapter 59. Writing a Custom Scan Provider</th></tr><tr><td width="10%" align="left"><a href="https://www.postgresql.org/docs/17/tablesample-support-functions.html">Prev</a></td><td width="10%" align="left"><a href="https://www.postgresql.org/docs/17/internals.html">Up</a></td><th width="60%" align="center">Part VII. Internals</th><td width="10%" align="right"></td><td width="10%" align="right"><a href="https://www.postgresql.org/docs/17/custom-scan-path.html">Next</a></td></tr></tbody></table>

---

PostgreSQL supports a set of experimental facilities which are intended to ==allow extension modules to add new scan types to the system==. Unlike a [foreign data wrapper](https://www.postgresql.org/docs/17/fdwhandler.html "Chapter 57. Writing a Foreign Data Wrapper"), which is only responsible for knowing how to scan its own foreign tables, a custom scan provider can provide an alternative method of scanning any relation in the system. Typically, the motivation for writing a custom scan provider will be to allow the use of some optimization not supported by the core system, such as caching or some form of hardware acceleration. This chapter outlines how to write a new custom scan provider.

Implementing ==a new type of custom scan is a three-step process==. First, during planning, it is necessary to generate access paths representing a scan using the proposed strategy. Second, if one of those access paths is selected by the planner as the optimal strategy for scanning a particular relation, the access path must be converted to a plan. Finally, it must be possible to execute the plan and generate the same results that would have been generated for any other access path targeting the same relation.

## Submit correction

If you see anything in the documentation that is not correct, does not match your experience with the particular feature or requires further clarification, please use [this form](https://www.postgresql.org/account/comments/new/17/custom-scan.html/) to report a documentation issue.