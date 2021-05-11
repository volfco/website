---
title: "Exploring Clickhouse Users"
date: 2018-09-29T11:36:33+08:00
draft: true
tags: 
  - Clickhouse
---

Clickhouse is a nifty pice of software.

The only other DB I know that can provide row level security- PostgreSQL. I'm sure there are others, but I personally don't have experience with them.

## Users Overview
Recently, Clickhouse introduced a new interface to user management. You can execute user modification commands via SQL and DDL statements- before it was limited to modification to users.xml. 

**The thing that should be noted is that once a resource exists, the default changes from open to closed.** If you have no row level permissions, all users have all permissions. Once you add row permissions, all other users have no permissions. 

## The General Idea
Invision this- You have an application that is multi-tenant, say a DNS Provider, that generates metrics. Normal convention would tell us that we would build an application that does the following:
1. Looks up the session token and pulls account information, in our example it's Domains attached to the User's account
2. Build a Query based on the provided data and the requested endpoint
3. Execute the query
4. Return data to the user

Now this isn't horrible, it's good even. But what if we don't want to build this service. 

## Building permissions


## Updating