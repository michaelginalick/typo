---
title: "Opensearch Explorations"
date: "2024-11-20T09:36:03-06:00"
summary: "Navigating software features and functionality"
description: "Navigating software features and functionality"
toc: false
readTime: true
autonumber: true
math: false
tags: ["AWS Opensearch"]
showTags: true
hideBackToTop: false
---

The team I’m working on is in the process of reinventing how users are presented content on our site. This includes features like search and determining what items users see on the landing page or subsequent pages. Currently, the primary way users come to the site is via email. The content in these emails is configurable at an individual level. For the entire existence of the company, this has been the main focus. If enough data is collected, the data processes are smart enough to serve individualized content.

To put it more plainly, the content in the email I receive may not be the content in your email. By all accounts, this approach works. The company generates enough user engagement through email to be profitable. However, the drawback of this approach is that once a user clicks through to the site, the content they see is mostly static. By static, I mean there isn’t much intelligence on the site itself. For example, most users see the same content in the “you may also like” section. For the most part, content is served based on recency. Given the ephemeral nature of the market the company operates in, this makes sense. Content expires, and it wouldn’t provide a good user experience if users were shown expired content.

# The Problem

We store content in AWS OpenSearch. As far as I can tell, OpenSearch is just Elasticsearch under a different name. In fact, whenever I need to research something for OpenSearch, I usually just Google Elasticsearch and can find what I’m looking for.

The business team approached us with a request: they wanted to sort content on the site based on popularity. Popularity, in this context, is simply user engagement—or, more simply, clicks. They wanted to measure this across three timeframes: 72 hours, 30 days, and 90 days.

# Developing the Feature

With the requirements in hand, I started developing the feature. I’ll spare you the details here, but the first step was integrating with a third-party system that tracks user engagement. This involved setting up an SQS queue and ingesting the data into a separate index. The next step was taking the data in this new index (let’s call it “popularity”) and hydrating the documents in another index with the popularity data. This was straightforward and involved setting up a few Cron jobs (or, in AWS terms, CloudWatch Events). An event would trigger a Lambda function that queried one index and wrote the results to another.

This setup worked well for a few weeks—until the amount of data being queried became non-trivial.

The query we were using was something OpenSearch warns against. In plain terms, it was simply counting how many times an attribute occurred within a given time frame. Here’s an example of the query:

```json
{
  "size": 0,
  "query": {
    "bool": {
      "filter": {
        "term": {
          "detail.properties.post_uid": [uids...]
        }
      }
    }
  },
  "aggs": {
    "uids": {
      "terms": {
        "field": "detail.properties.post_uid.keyword",
        "size": 10
      },
      "aggs": {
        "clicks_last_90_days": {
          "date_range": {
            "field": "detail.timestamp",
            "ranges": [
              { "from": "now-90d/d", "to": "now" },
              { "from": "now-30d/d", "to": "now" },
              { "from": "now-72h/h", "to": "now" }
            ]
          }
        }
      }
    }
  }

}
```
This query caused massive CPU spikes in OpenSearch whenever it was run. We attempted several fixes, such as batching the number of items being submitted to the query. While this slightly improved performance, it didn’t solve the underlying issue.

# A Better Approach

At this point, you might be wondering, “Isn’t this relational data? Why not use a relational database?” You’re not wrong. However, in many organizations, the tools you have to work with are often dictated by constraints. In this case, OpenSearch was the tool already in use.

This approach had two primary issues:

1. The CPU spikes, which put the cluster at risk of crashing.
2. Redundant queries. The same data was being queried unnecessarily. For example, as content expired or saw reduced engagement, the same large queries were being issued. Instead of counting each item every time the query ran, it would have been better to maintain a running tally of engagements as they were processed.


# The Solution: Index Transformations

After extensive research and trial and error, I discovered a promising feature, [Index transformations](https://opensearch.org/docs/latest/im-plugin/index-transforms/index/). The OpenSearch documentation provides an excellent explanation of this feature’s use cases, so I won’t go into too much detail. In summary, it was exactly what I needed. Instead of issuing the same query for every document, the transform job runs continuously and stores the results for each document.

Here’s an example of the transform job:

```json
{
  "transform": {
    "description": "Track click events per post_uid in 72 hours, 30 days, and 90 days intervals",
    "page_size": 10,
    "enabled": true,
    "continuous": true,
    "schedule": {
      "interval": {
        "period": 5,
        "unit": "Minutes"
      }
    },
    "source_index": "click-events*",
    "target_index": "post-click-events",
    "data_selection_query": {
      "exists": {
        "field": "detail.properties.post_uid.keyword"
      }
    },
    "groups": [
      {
        "terms": {
          "source_field": "detail.properties.post_uid.keyword"
        }
      }
    ],

    "aggregations": {
      "clicks_per_period": {
        "scripted_metric": {
          "init_script": "state.clicks_90d = 0; state.clicks_30d = 0; state.clicks_72h = 0;",
          "map_script": """
            def timestamp = doc['detail.timestamp'].value.toInstant();

            Instant now = Instant.ofEpochMilli(System.currentTimeMillis());
            if (timestamp.isAfter(now.minusSeconds(90L * 24 * 60 * 60))) {
              state.clicks_90d += 1;
            }
            if (timestamp.isAfter(now.minusSeconds(30L * 24 * 60 * 60))) {
              state.clicks_30d += 1;
            }
            if (timestamp.isAfter(now.minusSeconds(72L * 60 * 60))) {
              state.clicks_72h += 1;
            }
          """,
          "combine_script": "return state;",
          "reduce_script": """
            Map result = new HashMap();
            result.clicks_90d = 0;
            result.clicks_30d = 0;
            result.clicks_72h = 0;
            for (s in states) {
              result.clicks_90d += s.clicks_90d;
              result.clicks_30d += s.clicks_30d;
              result.clicks_72h += s.clicks_72h;
            }
            return result;
          """
        }
      }
    }
  }
}

```
Obviously this code is much more complex, and took awhile to get correct. However, it was worth the effort. Moving to this approach eliminated the need to query over massive amounts of data, and as a result we saw the CPU usage return to normal. I should note that when we first ran this job, CPU levels spiked. This is because the job needed to go back through every document in the index to get the inital counts. Once the job completed the initial sweep, CPU usage went back down.

# The Lesson
Many times the initial solution to something does not hold up in the long term. I spent a lot of time attempting to improve the initial query with the hopes that it would work. It was only until I gave up with one approach was I able to obtain a more complete solution.
