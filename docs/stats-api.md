---
title: Stats API reference
---

import {Required, Optional} from '../src/js/api-helpers.js';


The Plausible Stats API offers a way to retrieve your stats programmatically. It's a read-only interface to present historical and real-time stats only. Take a look at our [events API](events-api.md) if you want to send pageviews or custom events to our backend and our [sites API](sites-api.md) if you want to manage your sites through the API.

The API accepts GET requests with query parameters and returns standard HTTP responses along with a JSON-encoded body. All API requests must be made over HTTPS. Calls made over plain HTTP will fail. API requests without authentication will also fail.

Each request must be authenticated with an API key using the Bearer Token method. You can obtain an API key for your account by going to your user
settings page [plausible.io/settings](https://plausible.io/settings).

API keys have a rate limit of 600 requests per hour by default. If you have special needs for more requests, please contact us to request more capacity.

The easiest way to explore the API is by using our Postman collection. Just define your `TOKEN` and `SITE_ID` variables, and you'll have an executable API reference ready to go.

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/32c111c4f6d2cccef9dd)

:::note
New to Postman? We have [a step-by-step guide to set up the authorization and run your first queries with Postman](get-started-with-postman.md).
:::

## Concepts

Querying the Plausible API will feel familiar if you have used time-series databases before. You can't query individual records from
our stats database. You can only request aggregated metrics over a certain time period.

Each request requires a `site_id` parameter which is the domain of your site as configured in Plausible. If you're unsure, navigate to your site
settings in Plausible and grab the value of the `domain` field.

### Metrics

You can specify a `metrics` option in the query, to choose the metrics for each instance returned. See here for a full overview of [metrics and their definitions](/metrics-definitions). The metrics currently supported in Stats API are:

| Metric            | Description                                                                                                                                               |
|-------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| `visitors`        | The number of unique visitors.                                                                                                                            |
| `visits`          | The number of visits/sessions                                                                                                                             |
| `pageviews`       | The number of pageview events                                                                                                                             |
| `views_per_visit` | The number of pageviews divided by the number of visits. Returns a floating point number. currently only supported in Aggregate and Timeseries endpoints. |
| `bounce_rate`     | Bounce rate percentage                                                                                                                                    |
| `visit_duration`  | Visit duration in seconds                                                                                                                                 |
| `events`          | The number of events (pageviews + custom events)                                                                                                          |

### Time periods

The options are identical for each endpoint that supports configurable time periods. Each period is relative to a `date` parameter. The date should follow the standard ISO-8601 format. When not specified, the `date` field defaults to `today(site.timezone)`.
All time calculations on our backend are done in the time zone that the site is configured in.

* `12mo,6mo` - Last n calendar months relative to `date`.
* `month` - The calendar month that `date` falls into.
* `30d,7d` - Last n days relative to `date`.
* `day` - Stats for the full day specified in `date`.
* `custom` - Provide a custom range in the `date` parameter.

When using a custom range, the `date` parameter expects two ISO-8601 formatted dates joined with a comma as follows `?period=custom&date=2021-01-01,2021-01-31`.
Stats will be returned for the whole date range inclusive of the start and end dates.

### Properties

Each pageview and custom event in our database has some predefined _properties_ associated with it. In other analytics tools, these
are often referred to as _dimensions_ as well. Properties can be used for filtering and breaking down your stats to drill into
more depth. Here's the full list of properties we collect automatically:

| Property              | Example                       | Description                                                                                                                             |
| --------------------- | ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| event:name            | pageview                      | Name of the event triggered. `pageview` is a reserved event name but custom events can be named anything. Use this property to represent a custom event goal.                             |
| event:page            | /blog/remove-google-analytics | Pathname of the page where the event is triggered. You can also use wildcards to match with multiple paths. See [here](/pageview-goals#group-your-pages-using-wildcards) for examples.  |
| visit:entry_page      | /home                         | Page on which the visit session started (landing page).                                                                                 |
| visit:exit_page       | /home                         | Page on which the visit session ended (last page viewed).                                                                               |
| visit:source          | Twitter                       | Visit source, populated from an url query parameter tag (`utm_source`, `source` or `ref`) or the `Referer` HTTP header.                 |
| visit:referrer        | t.co/fzWTE9OTPt               | Raw `Referer` header without `http://`, `http://` or `www.`.                                                                            |
| visit:utm_medium      | social                        | Raw value of the `utm_medium` query param on the entry page.                                                                            |
| visit:utm_source      | twitter                       | Raw value of the `utm_source` query param on the entry page.                                                                            |
| visit:utm_campaign    | profile                       | Raw value of the `utm_campaign` query param on the entry page.                                                                          |
| visit:utm_content     | banner                        | Raw value of the `utm_content` query param on the entry page.                                                                           |
| visit:utm_term        | keyword                       | Raw value of the `utm_term` query param on the entry page.                                                                              |
| visit:device          | Desktop                       | Device type. Possible values are `Desktop`, `Laptop`, `Tablet` and `Mobile`.                                                            |
| visit:browser         | Chrome                        | Name of the browser vendor. Most popular ones are `Chrome`, `Safari` and `Firefox`.                                                     |
| visit:browser_version | 88.0.4324.146                 | Version number of the browser used by the visitor.                                                                                      |
| visit:os              | Mac                           | Name of the operating system. Most popular ones are `Mac`, `Windows`, `iOS` and `Android`. Linux distributions are reported separately. |
| visit:os_version      | 10.6                          | Version number of the operating system used by the visitor.                                                                             |
| visit:country         | US                            | ISO 3166-1 alpha-2 code of the visitor country.                                                                                         |
| visit:region          | US-MD                         | ISO 3166-2 code of the visitor region.                                                                                                  |
| visit:city            | 4347778                       | [GeoName ID](https://www.geonames.org/) of the visitor city.                                                                            |

#### Custom properties

In addition to properties that are collected automatically, you can also query for [custom properties](/custom-props/introduction).
To filter or break down by a custom property, use the key `event:props:<custom_prop_name>`. [See example](#breakdown-custom-event-by-custom-props) for how to use it.

:::note
Currently clients are limited to filtering or breaking down on just one custom property at a time. Custom property filters and breakdowns can't be combined arbitrarily. We are aware of this issue and we have plans to fix it, but we rely on our database to support some new features to fix this issue.
:::

### Filtering

Most endpoints support a `filters` query parameter to drill down into your data. Currently, only simple equality filters are supported.

An equality filter can be specified with url-encoded `==`. Filters can be joined together with `;` which applies a logical
`AND` operator to the filters. Here's a filter expression combining two filters:

```
visit:browser==Firefox;visit:country==FR
```

You can join values together with a `|` to express an IN filter. The filter will match if the key is
in any of the values. For example, the following filter:

```
visit:country==FR|DE
```

Would match both visitors from both France and Germany.

:::note
Want to use the `|` character in a filter value? You can escape it with a backslash. For example, `visit:utm_campaign==campaign\|one` will let you filter by the literal `campaign|one` value.
:::

You can also exclude by a specific property, using a `!=` filter:

```
visit:country!=FR
```

It is currently only possible to exclude one value at a time.

#### Filtering by pageview goals

[Pageview goals](/pageview-goals) are a convenient way to see pageviews for a specific page path in the Plausible dashboard. To filter by a pageview goal in Stats API, simply use a filter like this: `event:page==/your-page`. As in your dashboard, you can use [wildcards](/pageview-goals#pageview-goals-support-wildcards) like `event:page==/blog**` to represent pageview goals that match with multiple paths.  

## Endpoints

### GET /api/v1/stats/realtime/visitors

This endpoint returns the number of current visitors on your site. A current visitor is defined as a visitor who triggered a pageview on your site
in the last 5 minutes.

```bash title="Try it yourself"
curl 'https://plausible.io/api/v1/stats/realtime/visitors?site_id=$SITE_ID'
  -H "Authorization: Bearer ${TOKEN}"
```

```json title="Response"
21
```

#### Parameters
<hr / >

**site_id** <Required />

Domain of your site on Plausible.
<hr / >

### GET /api/v1/stats/aggregate

This endpoint aggregates metrics over a certain time period. If you are familiar with the Plausible dashboard, this endpoint corresponds to the top row of stats that include `Unique Visitors` Pageviews, `Bounce rate` and `Visit duration`. You can retrieve any number and combination of these metrics in one request.


```bash title="Try it yourself"
curl 'https://plausible.io/api/v1/stats/aggregate?site_id=$SITE_ID&period=6mo&metrics=visitors,pageviews,bounce_rate,visit_duration' \
  -H "Authorization: Bearer ${TOKEN}"
```

```json title="Response"
{
  "results": {
    "bounce_rate": {
        "value": 53.0
    },
    "pageviews": {
        "value": 673814
    },
    "visit_duration": {
        "value": 86.0
    },
    "visitors": {
        "value": 201524
    }
  }
}
```


#### Parameters
<hr / >

**site_id** <Required />

Domain of your site on Plausible.

<hr / >

**period** <Optional />

See [time periods](#time-periods). If not specified, it will default to `30d`.

<hr / >

**metrics** <Optional />

List of metrics to aggregate. Valid options are `visitors`, `visits`, `pageviews`, `views_per_visit`, `bounce_rate`, `visit_duration` and `events`. If not specified, it will default to `visitors`.

<hr / >

**compare** <Optional />

Off by default. You can specify `compare=previous_period` to calculate the percent difference with the previous period for each metric. The previous period will be of the exact same length as specified in the `period` parameter.

<hr / >

**filters** <Optional />

See [filtering](#filtering)


### GET /api/v1/stats/timeseries

This endpoint provides timeseries data over a certain time period. If you are familiar with the Plausible dashboard, this endpoint corresponds to
the main visitor graph.


```bash title="Try it yourself"
curl 'https://plausible.io/api/v1/stats/timeseries?site_id=$SITE_ID&period=6mo'
  -H "Authorization: Bearer ${TOKEN}"
```

```json title="Response"
{
  "results": [
    {
      "date": "2020-08-01",
      "visitors": 36085
    },
    {
      "date": "2020-09-01",
      "visitors": 27688
    },
    {
      "date": "2020-10-01",
      "visitors": 71615
    },
    {
      "date": "2020-11-01",
      "visitors": 31440
    },
    {
      "date": "2020-12-01",
      "visitors": 35804
    },
    {
      "date": "2021-01-01",
      "visitors": 0
    }
  ]
}
```


#### Parameters
<hr / >

**site_id** <Required />

Domain of your site on Plausible.

<hr / >

**period** <Optional />

See [time periods](#time-periods). If not specified, it will default to `30d`.

<hr / >

**filters** <Optional />

See [filtering](#filtering)

<hr / >

**metrics** <Optional />

Comma-separated list of metrics to show for each time bucket. Valid options are `visitors`, `visits`, `pageviews`, `views_per_visit`, `bounce_rate` and `visit_duration`. If not
specified, it will default to `visitors`.

<hr / >

**interval** <Optional />

Choose your reporting interval. Valid options are `date` (always) and `month` (when specified period is longer than one calendar month). Defaults to
`month` for `6mo` and `12mo`, otherwise falls back to `date`.

### GET /api/v1/stats/breakdown

This endpoint allows you to break down your stats by some property. If you are familiar with SQL family databases, this endpoint corresponds to
running `GROUP BY` on a certain property in your stats, then ordering by the count.

Check out the [properties](#properties) section for a reference of all the properties you can use in this query.

This endpoint can be used to fetch data for `Top sources`, `Top pages`, `Top countries` and similar reports.


```bash title="Try it yourself"
curl 'https://plausible.io/api/v1/stats/breakdown?site_id=$SITE_ID&period=6mo&property=visit:source&metrics=visitors,bounce_rate&limit=5' \
  -H "Authorization: Bearer ${TOKEN}"
```

```json title="Response"
{
  "results": [
    {
        "bounce_rate": 49.0,
        "source": "(Direct / None)",
        "visitors": 94932
    },
    {
        "bounce_rate": 75.0,
        "source": "Hacker News",
        "visitors": 22540
    },
    {
        "bounce_rate": 58.0,
        "source": "Google",
        "visitors": 16909
    },
    {
        "bounce_rate": 62.0,
        "source": "Twitter",
        "visitors": 7477
    },
    {
        "bounce_rate": 56.0,
        "source": "indiehackers.com",
        "visitors": 4518
    }
  ]
}
```

#### Parameters
<hr / >

**site_id** <Required />

Domain of your site on Plausible.

<hr / >

**property** <Required />

Which property to break down the stats by. Valid options are listed in the [properties](#properties) section above.

<hr / >

**period** <Optional />

See [time periods](#time-periods). If not specified, it will default to `30d`.

<hr / >

**metrics** <Optional />

Comma-separated list of metrics to show for each item in breakdown. Valid options are `visitors`, `pageviews`, `bounce_rate`, `visit_duration`, `visits` and `events`. If not
specified, it will default to `visitors`.

<hr / >

**limit** <Optional />

Limit the number of results. Maximum value is 1000. Defaults to 100. If you want to get more than 1000 results, you can make multiple requests and paginate the results by specifying the `page` parameter (e.g. make the same request with `page=1`, then `page=2`, etc)

**page** <Optional />

Number of the page, used to paginate results. Importantly, the page numbers start from 1 not 0.

**filters** <Optional />

See [filtering](#filtering)

#### Breaking down by multiple properties at the same time

Currently, it is only possible to break down on one property at a time. Using a list of properties with one query is not supported. So if you want a breakdown by both `event:page` and `visit:source` for example, you would have to make multiple queries (break down on one property and filter on another) and then manually/programmatically group the results together in one report. This also applies for breaking down by time periods. To get a daily breakdown for every page, you would have to break down on `event:page` and make multiple queries for each date. For a simple time period breakdown, have a look at the Timeseries endpoint.

## Examples of common queries

### Top pages

Let's say you want to show a similar report to the `Top pages` report in the Plausible UI. You can do this by calling the
`/api/v1/stats/breakdown` endpoint and specify `event:page` as the property to group by.

```bash title="Top pages"
curl 'https://plausible.io/api/v1/stats/breakdown?site_id=$SITE_ID&period=6mo&property=event:page&limit=5'
  -H "Authorization: Bearer ${TOKEN}"
```

```json title="Response"
{
  "results": [
    {
      "page": "/",
      "visitors": 94298
    },
    {
      "page": "/blog/open-source-licenses",
      "visitors": 18803
    },
    {
      "page": "/plausible.io",
      "visitors": 20485
    },
    {
      "page": "/self-hosted-web-analytics",
      "visitors": 22236
    },
    {
      "page": "/sites",
      "visitors": 32386
    }
  ]
}
```

### Number of visitors to a specific page

Let's say you want to get the number of visitors to a specific page on your website like `/order/confirmation`. This can be achieved by
filtering your stats on the `event:page` property:

```bash title="Visitors to /order/confirmation"
curl 'https://plausible.io/api/v1/stats/aggregate?site_id=$SITE_ID&period=6mo&filters=event:page%3D%3D%2Forder%2Fconfirmation'
  -H "Authorization: Bearer ${TOKEN}"
```

```json title="Response"
{
  "results": {
    "visitors": {
        "value": 1480
    }
  }
}
```

### Monthly traffic from Google

As a second example, let's imagine we want to analyze our SEO efforts for the last half year.
To graph your traffic from Google over time, you can use the `timeseries` endpoint with a time period `6mo` and
filter expression `visit:source==Google`.


```bash title="Monthly traffic from Google"
curl 'https://plausible.io/api/v1/stats/timeseries?site_id=$SITE_ID&period=6mo&filters=visit:source%3D%3DGoogle' \
  -H "Authorization: Bearer ${TOKEN}"
```

```json title="Response"
{
  "results": [
    {
        "date": "2020-09-01",
        "visitors": 2962
    },
    {
        "date": "2020-10-01",
        "visitors": 4974
    },
    {
        "date": "2020-11-01",
        "visitors": 5119
    },
    {
        "date": "2020-12-01",
        "visitors": 5397
    },
    {
        "date": "2021-01-01",
        "visitors": 7167
    },
    {
        "date": "2021-02-01",
        "visitors": 5802
    }
]
}
```

### Breakdown custom event by custom properties

A more advanced use-case where custom events are used along with custom properties. Let's say you have a `Download` custom event along with
a custom property called `method`. You can get a breakdown of download methods with the following query:

```bash title="Breakdown of download methods"
curl 'https://plausible.io/api/v1/stats/breakdown?site_id=$SITE_ID&period=6mo&property=event:props:method&filters=event:name%3D%3DDownload'
  -H "Authorization: Bearer ${TOKEN}"
```

```json title="Response"
{
    "results": [
        {
            "method": "HTTP",
            "visitors": 1477
        },
        {
            "method": "Magnet",
            "visitors": 370
        }
    ]
}
```

:::note
You can use GET https://plausible.io/api/health endpoint to monitor the status of our API
:::
