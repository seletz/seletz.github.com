---
layout:    post
title:    PostgreSQL, JSON Queries and MinIO events
date:     2023-11-16
comments: true
tags: postgresql programming
---


Yesterday I wanted to query several keys nested deeply in a JSON stored in a
PostgreSQL JSONB column.   The table -- DDL below -- is used by [MinIO](http://min.io) to add
events for subscribed S3 buckets -- more on that in another post.

```sql
create table public.minio_event_access
(
    event_time timestamp with time zone not null,
    event_data jsonb
);
```

![MinIO event table format](/assets/images/minio-event-access-table-format.png)

The JSON stored in the `event_data` column looks something like this:

```json
{
  "Records": [
    {
      "s3": {
        "bucket": {
          "arn": "arn:aws:s3:::berichtswesen",
          "name": "berichtswesen",
          "ownerIdentity": {
            "principalId": "minioadmin"
          }
        },
        "object": {
          "key": "bayern%2FAOK_Bayern_dev.pdf",
          "sequencer": "1797D2FD70159CB8",
          "versionId": "11ca60a8-59ff-4f44-823e-1a07a19fc3eb"
        },
        "configurationId": "Config",
        "s3SchemaVersion": "1.0"
      },
      "source": {
        "host": "127.0.0.1",
        "port": "",
        "userAgent": "MinIO (darwin; amd64) minio-go/v7.0.64 MinIO Console/(dev)"
      },
      "awsRegion": "",
      "eventName": "s3:ObjectRemoved:DeleteMarkerCreated",
      "eventTime": "2023-11-15T14:42:50.304Z",
      "eventSource": "minio:s3",
      "eventVersion": "2.0",
      "userIdentity": {
        "principalId": "minioadmin"
      },
      "responseElements": {
        "x-amz-id-2": "dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8",
        "content-length": "1537",
        "x-amz-request-id": "1797D2FD6FA840A0",
        "x-minio-deployment-id": "201361b0-c48a-4765-863d-476075968048",
        "x-minio-origin-endpoint": "http://192.168.200.140:9000"
      },
      "requestParameters": {
        "region": "",
        "principalId": "minioadmin",
        "sourceIPAddress": "127.0.0.1"
      }
    }
  ]
}
```

Now I wanted to select the object path and the event name, as well as the timestamp.  One way for doing this is to use the `#>>` [operator](https://www.postgresql.org/docs/current/functions-json.html):

```sql
select event_time,
       event_data #>> '{Records,0,eventName}'     as event_name,
       event_data #>> '{Records,0,s3,object,key}' as path
from minio_event_access
order by event_time;
```

This -- as stated in the docs -- results in *text values*:

| event\_time | event\_name | path |
| :--- | :--- | :--- |
| 2023-11-15 15:05:19.876000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern.pdf |
| 2023-11-15 15:05:19.877000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_Scan\_1.pdf |
| 2023-11-15 15:05:19.877000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern.pdf |
| 2023-11-15 15:05:19.878000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_Scan\_2\_bend.pdf |
| 2023-11-15 15:05:19.878000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_Scan\_1.pdf |
| 2023-11-15 15:05:19.879000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_Scan\_3\_defaced.pdf |
| 2023-11-15 15:05:19.879000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_Scan\_2\_bend.pdf |
| 2023-11-15 15:05:19.880000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_Scan\_3\_defaced.pdf |
| 2023-11-15 15:05:19.881000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_Scan\_4\_b\_n\_w.pdf |
| 2023-11-15 15:05:19.881000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_Scan\_4\_b\_n\_w.pdf |
| 2023-11-15 15:05:19.882000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_Scan\_5\_bad.pdf |
| 2023-11-15 15:05:19.882000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_Scan\_5\_bad.pdf |
| 2023-11-15 15:05:19.883000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_dev.pdf |
| 2023-11-15 15:05:19.883000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_dev.pdf |
| 2023-11-15 15:05:19.884000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_dev\_Scan\_1\_manipulated.pdf |
| 2023-11-15 15:05:19.885000 +00:00 | s3:ObjectRemoved:Delete | bayern%2FAOK\_Bayern\_dev\_Scan\_1\_manipulated.pdf |
| 2023-11-15 15:08:22.460000 +00:00 | s3:ObjectCreated:Put | assets%2Fpdf%2Faok%2Fbw%2FAOK\_BW\_dev.pdf |
| 2023-11-15 15:08:22.464000 +00:00 | s3:ObjectCreated:Put | assets%2Fpdf%2Faok%2Fbw%2FAOK\_BW.pdf |
| 2023-11-15 15:08:22.468000 +00:00 | s3:ObjectCreated:Put | assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_dev.pdf |
| 2023-11-15 15:08:22.469000 +00:00 | s3:ObjectCreated:Put | assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern.pdf |
| 2023-11-15 15:08:22.489000 +00:00 | s3:ObjectCreated:Put | assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_dev\_Scan\_1\_manipulated.pdf |
| 2023-11-15 15:08:22.495000 +00:00 | s3:ObjectCreated:Put | assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_Scan\_4\_b\_n\_w.pdf |
| 2023-11-15 15:08:22.501000 +00:00 | s3:ObjectCreated:Put | assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_Scan\_1.pdf |
| 2023-11-15 15:08:22.514000 +00:00 | s3:ObjectCreated:Put | assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_Scan\_2\_bend.pdf |
| 2023-11-15 15:08:22.514000 +00:00 | s3:ObjectCreated:Put | assets%2Flogo%2Ficon-iconmonstr-file-37.svg |
| 2023-11-15 15:08:22.523000 +00:00 | s3:ObjectCreated:Put | assets%2Flogo%2FRevisor.svg |
| 2023-11-15 15:08:22.526000 +00:00 | s3:ObjectCreated:Put | assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_Scan\_5\_bad.pdf |
| 2023-11-15 15:08:22.530000 +00:00 | s3:ObjectCreated:Put | assets%2Flogo%2FRevisor.png |
| 2023-11-15 15:08:22.534000 +00:00 | s3:ObjectCreated:Put | assets%2Flogo%2FRevisor.curve |
| 2023-11-15 15:08:22.551000 +00:00 | s3:ObjectCreated:Put | assets%2Fapi%2Fcalculus.yml |
| 2023-11-15 15:08:22.552000 +00:00 | s3:ObjectCreated:Put | assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_Scan\_3\_defaced.pdf |

| 2023-11-15 15:08:22.555000 +00:00 | s3:ObjectCreated:Put | assets%2Fapi%2Fcutomercare.yml |

One can [*also* use](https://www.postgresql.org/docs/current/functions-json.html) `json_path_query()`:

```sql

select event_time,
       jsonb_path_query(event_data, '$.Records[0].eventName')     as event_name,
       jsonb_path_query(event_data, '$.Records[0].s3.object.key') as path
from minio_event_access
order by event_time;
```

Which results in **JSON** values:

| event\_time | event\_name | path |
| :--- | :--- | :--- |
| 2023-11-15 15:05:19.885000 +00:00 | "s3:ObjectRemoved:Delete" | "bayern%2FAOK\_Bayern\_dev\_Scan\_1\_manipulated.pdf" |
| 2023-11-15 15:08:22.460000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fpdf%2Faok%2Fbw%2FAOK\_BW\_dev.pdf" |
| 2023-11-15 15:08:22.464000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fpdf%2Faok%2Fbw%2FAOK\_BW.pdf" |
| 2023-11-15 15:08:22.468000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_dev.pdf" |
| 2023-11-15 15:08:22.469000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern.pdf" |
| 2023-11-15 15:08:22.489000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_dev\_Scan\_1\_manipulated.pdf" |
| 2023-11-15 15:08:22.495000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_Scan\_4\_b\_n\_w.pdf" |
| 2023-11-15 15:08:22.501000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_Scan\_1.pdf" |
| 2023-11-15 15:08:22.514000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_Scan\_2\_bend.pdf" |
| 2023-11-15 15:08:22.514000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Flogo%2Ficon-iconmonstr-file-37.svg" |
| 2023-11-15 15:08:22.523000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Flogo%2FRevisor.svg" |
| 2023-11-15 15:08:22.526000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_Scan\_5\_bad.pdf" |
| 2023-11-15 15:08:22.530000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Flogo%2FRevisor.png" |
| 2023-11-15 15:08:22.534000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Flogo%2FRevisor.curve" |
| 2023-11-15 15:08:22.551000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fapi%2Fcalculus.yml" |
| 2023-11-15 15:08:22.552000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fpdf%2Faok%2Fbayern%2FAOK\_Bayern\_Scan\_3\_defaced.pdf" |
| 2023-11-15 15:08:22.555000 +00:00 | "s3:ObjectCreated:Put" | "assets%2Fapi%2Fcutomercare.yml" |
