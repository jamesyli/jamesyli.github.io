---
layout: post
title: "Performance analysis of Designate /v2/recordsets API"
---

## Introduction

The new Designate API `/v2/recordsets` was found slow in dealing with a large number of recordsets in database. A profiling shows that two DB operations are the main contributors to the long response time.

- Recordsets Sorting

   When retrieving matched recordsets Designate shows them in a specific order (`created_at` by default) which is accomplished by a `ORDER BY` sql statement. If a proper DB index is missing or not chosen the sql command runs slowly. Please be aware Designate supports various sort keys like `updated_at`, `zone_id`, `tenant_id`, `name`, `ttl`, `type`.

- Recordsets couting

   Designate shows a `total_count` metadata in each `recordsets` request. If a user who owns 1M recordsets wants to do a matching like `/v2/recordsets?name=*foo*`, the count sql query is observed slow.

## Proposed methods

The patch [#328813](https://review.openstack.org/#/c/328813) proposes ways to solve the issues.

- Added **five** new DB indexes and ensures mysql chooses the proper one when doing recordsets sorting with various sort keys.

  *Needs to evaluate how new indexes impact DB writes*
   
- Introduce a new header for operator/customer to turn off recordset counting

## Tests

Tests were conducted to evaluate the proposed methods. We focus on:

- How much performance gain on recordsets listing/filtering

- How much impact on zones/recordsets creation

### Environment

We loaded a testing environment at Rackspace with `208,480` zones and `1,442,265` recordsets in DB. The tenant as which our tests ran has `A` zones and `B` recordsets. The tests create 100 zones, 500 recordsets, and do recordsets filtering 100 times, and measure response time for each request.

## Results

|                                      |    Baseline (w/o patch #328813)   |    w/ patch #328813    |
| ------------------------------------ | --------------------------------- | ---------------------- |
| GET /recordsets?data=%random digits% |                                   |                        |


## Discussions

