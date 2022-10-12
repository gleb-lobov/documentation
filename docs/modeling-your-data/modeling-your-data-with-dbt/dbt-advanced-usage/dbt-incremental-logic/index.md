---
title: "Incremental Logic"
date: "2022-10-05"
sidebar_position: 300
---
```mdx-code-block
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
```

The general principle behind an incremental model is to identify new events/rows since the previous run of the model, and then only process these new events. This minimizes cost and reduces run times.

For mobile and web event data we typically consider a session to be a complete 'visit' and as such calculate metrics across the entire session. This means that when we have a new event for a previously processed session, we have to reprocess all historic events for that session as well as the new events. The logic followed is:

1. Identify new events since the previous run of the package.
2. Identify the `session_id` associated with the new events.
3. Look back over the events table to find all events associated with these `sessions_id`.
4. Run all these events through the page/screen views, sessions and users modules.

Given the large nature of event tables, Step 3 can be an expensive operation. To minimize cost ideally we want to:

- Know when any given session started. This would allow us to limit scans on the events table when looking back for previous events.
  - This is achieved by the `snowplow_web/mobile_base_sessions_lifecycle_manifest` model, which records the start and end timestamp of all sessions.
- Limit the maximum allowed session length. Sessions generated by bots can persist for years. This would mean scanning years of data every run of the package.
  - For the web package this is achieved by the `snowplow_web_base_quarantined_sessions` model, which stores the `session_id` of any sessions that have exceeded the max allowed session length (`snowplow__max_session_days`). For such sessions, all events are processed up until the max allowed length. Moving forward, no more data is processed for that session. This is not required for mobile.

## The Incremental Manifest

The web and mobile packages use centralized manifest tables, `snowplow_web/mobile_incremental_manifest`, to record what events have already been processed and by which model/node. This allows for easy identification of what events to process in subsequent runs of the package. The manifest table is updated as part of an `on-run-end` hook, which calls the `snowplow_incremental_post_hook()` macro.

<Tabs groupId="dbt-packages">
<TabItem value="web" label="Snowplow Web" default>

Example `snowplow_web_incremental_manifest`:

| model                            | last_success |
|----------------------------------|--------------|
| snowplow_web_page_views_this_run | '2021-06-03' |
| snowplow_web_page_views          | '2021-06-03' |
| snowplow_web_sessions            | '2021-06-02' |

</TabItem>
<TabItem value="mobile" label="Snowplow Mobile">

Example `snowplow_mobile_incremental_manifest`:

| model                                 | last_success |
| ------------------------------------- | ------------ |
| snowplow_mobile_screen_views_this_run | '2021-06-03' |
| snowplow_mobile_screen_views          | '2021-06-03' |
| snowplow_mobile_sessions              | '2021-06-02' |

</TabItem>
</Tabs>


## Identification of events to process

<Tabs groupId="dbt-packages">
<TabItem value="web" label="Snowplow Web" default>

The identification of which events to process is performed by the `get_run_limits` macro which is called in the `snowplow_web_base_new_event_limits` model. This macro uses the metadata recorded in `snowplow_web_incremental_manifest` to determine the correct events to process next based on the current state of the Snowplow dbt Web model. The selection of these events is done by specifying a range of `collector_tstamp`'s to process, between `lower_limit` and `upper_limit`. The calculation of these limits is as follows.


First we query `snowplow_web_incremental_manifest`, filtering for all enabled models tagged with `snowplow_web_incremental` within your dbt project:

```sql
select 
    min(last_success) as min_last_success,
    max(last_success) as max_last_success,
    coalesce(count(*), 0) as models
from snowplow_web_incremental_manifest
where model in (array_of_snowplow_tagged_enabled_models)
```
</TabItem>
<TabItem value="mobile" label="Snowplow Mobile">

The identification of which events to process is performed by the `get_run_limits` macro which is called in the `snowplow_mobile_base_new_event_limits` model. This macro uses the metadata recorded in `snowplow_mobile_incremental_manifest` to determine the correct events to process next based on the current state of the Snowplow dbt Mobile model. The selection of these events is done by specifying a range of `collector_tstamp`'s to process, between `lower_limit` and `upper_limit`. The calculation of these limits is as follows.

First we query `snowplow_mobile_incremental_manifest`, filtering for all enabled models tagged with `snowplow_mobile_incremental` within your dbt project:

```sql
select 
    min(last_success) as min_last_success,
    max(last_success) as max_last_success,
    coalesce(count(*), 0) as models
from snowplow_mobile_incremental_manifest
where model in (array_of_snowplow_tagged_enabled_models)
```

</TabItem>
</Tabs>

------

Based on the results the web model enters 1 of 4 states:

:::tip

In all states the `upper_limit` is limited by the `snowplow__backfill_limit_days` variable. This protects against back-fills with many rows causing very long run times.

:::
### State 1: First run of the package

The query returns `models = 0` indicating that no models exist in the manifest.

**`lower_limit`**: `snowplow__start_date`  
**`upper_limit`**: `least(current_tstamp, snowplow__start_date + snowplow__backfill_limit_days)`

### State 2: New model introduced

`models < size(array_of_snowplow_tagged_enabled_models)` and therefore a new model, tagged with `snowplow_web_incremental`, has been added since the last run. The package will replay all previously processed events in order to back-fill the new model.

**`lower_limit`**: `snowplow__start_date`  
**`upper_limit`**: `least(max_last_success, snowplow__start_date + snowplow__backfill_limit_days)`

### State 3: Models out of sync

`min_last_success < max_last_success` and therefore the tagged models are out of sync, for example due to a particular model failing to execute successfully during the previous run. The package will attempt to sync all models.

**`lower_limit`**: `min_last_success - snowplow__lookback_window_hours`  
**`upper_limit`**: `least(max_last_success, min_last_success + snowplow__backfill_limit_days)`

### State 4: Standard run

If none of the above criteria are met, then we consider it a 'standard run' and we carry on from the last processed event.

**`lower_limit`**: `max_last_success - snowplow__lookback_window_hours`  
**`upper_limit`**: `least(current_tstamp, max_last_success + snowplow__backfill_limit_days)`


### How to identify the current state

<Tabs groupId="dbt-packages">
<TabItem value="web" label="Snowplow Web" default>

If you want to check the current state of the web model, run the `snowplow_web_base_new_event_limits` model. This will log the current state to the CLI while causing no disruption to the incremental processing of events.

```bash
dbt run --select snowplow_web_base_new_event_limits
...
00:26:28 | 1 of 1 START table model scratch.snowplow_web_base_new_event_limits.. [RUN]
00:26:29 + Snowplow: Standard incremental run
00:26:29 + Snowplow: Processing data between 2021-01-05 17:59:32 and 2021-01-07 23:59:32
```

</TabItem>
<TabItem value="mobile" label="Snowplow Mobile">

If you want to check the current state of the mobile model, run the `snowplow_mobile_base_new_event_limits` model. This will log the current state to the CLI while causing no disruption to the incremental processing of events.

```bash
dbt run --select snowplow_mobile_base_new_event_limits
...
00:26:28 | 1 of 1 START table model scratch.snowplow_mobile_base_new_event_limits.. [RUN]
00:26:29 + Snowplow: Standard incremental run
00:26:29 + Snowplow: Processing data between 2021-01-05 17:59:32 and 2021-01-07 23:59:32
```

</TabItem>
</Tabs>