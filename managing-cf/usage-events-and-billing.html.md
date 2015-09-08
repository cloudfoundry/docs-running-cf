---
title: Usage Events and Billing
---

## Usage Events
Usage events can be used by Cloud Foundry operators to construct billing information for apps and service instances.

Despite being similar in name, usage events are different from Cloud Foundry audit events (`/v2/events`). The audit events stored in the events table should not be used for billing purposes; They are recorded regardless of the action succeeding and are not guaranteed to be in the correct order.

### App Usage Events

App Usage Events provide information about when users create, delete and update apps. They also include information about the app to enable operators to bill based on resource usage.

App Usage Events expire after 31 days by default. This is configurable using the cf-release manifest property `cc.app_usage_events.cutoff_age_in_days`.

#### Endpoint

`GET /v2/app_usage_events`

For documentation on the response information, see the [api docs](http://apidocs.cloudfoundry.org/latest-release/app_usage_events/list_all_app_usage_events.html)

#### Event States
App Usage Events have the following valid states: 'STARTED', 'STOPPED', and 'BUILDPACK_SET'.

Multiple 'STARTED' events can occur in a row, so as to indicate updates to the app - you can parse the response to determine the specific difference.  

Multiple 'STOPPED' events cannot occur without 'STARTED' events in-between them.  

The App Usage Events with the 'BUILDPACK_SET' state do not represent an actual application state; They instead signify that a buildpack has been used for the application. This can be used by operators who wish to charge for buildpack usage.


### Service Usage Events

Service Usage Events provide information about when users create, update, and delete service instances.

Service Usage Events currently do not expire like App Usage Events. The expiration of Service Usage Events will be implemented in a future Cloud Foundry release.

#### Endpoint

`GET /v2/service_usage_events`

For documentation on the response information, see the [api docs](http://apidocs.cloudfoundry.org/latest-release/service_usage_events/list_service_usage_events.html)

#### Event States

Service Usage Events have the following valid states: 'CREATED', 'DELETED', and 'UPDATED'.  These signify the desired state of the service.

#### Managed Service Instances vs User Provided Service Instances

Service Usage Events include the `service_instance_type` field to distinguish between managed service instances (service instances backed by a broker) and user provided service instances. You will likely not need to consider user provided service instances for billing purposes.

## Using Usage Events


### Ordering Usage Events
Order of events returned from the API are guaranteed to match the sequence of events that occurred in the system.

The timestamps on events should not be used to sequence events.  They may come from different cloud controller instances whose time could be slightly mismatched.  Unless you're billing on the millisecond, they will be precise enough for normal usage.

### Querying For Events
To only fetch events you have not yet seen, use the `after_guid` query parameter. This will only return usage events that occurred after the event with the provided guid.

Example:

```
GET /v2/app_usage_events?after_guid=7a63d717-3316-402f-bf12-a2a63211d1b9
```

Be aware that using the `after_guid` may result in lost events if you are not careful. Usage events are guaranteed to be in the correct order based on when the event creation started, but some event transactions take longer than others to complete.

If you always use the last event guid for `after_guid`, then you may miss events that occurred before that event, but that were still processing at the time of your query.

It is recommended that operators select their `after_guid` from an event far enough back in time to ensure that all hanging transactions finish.  The exact buffer period depends on the expected transaction time of a particular Cloud Foundry installation, but 1 minute is typically sufficient to prevent data loss.

### External Warehouse
To generate accurate billing information, Cloud Foundry operators will need to maintain their own, external data warehouses. These warehouses can persist App Usage Events and Service Usage Events for longer than their expiration periods and prevent operators from having to continuously make expensive usage event queries.

Commonly, operators generate aggregate billing events from the raw Cloud Foundry usage events to reduce the number of raw events they need to store.

### Creating Your Billing Epoch
When an operator first starts billing, there may be applications or instances without start events due to event expiration or other reasons. To seed the application usage events with a start event (epoch) for all applications use the purge endpoints.

|Event Type| Endpoint | Documentation |
|---|---|---|
| App Usage Events | `/v2/app_usage_events/destructively_purge_all_and_reseed_started_apps`| [api docs](http://apidocs.cloudfoundry.org/latest-release/service_usage_events/list_service_usage_events.html) |
| Service Usage Events | `/v2/service_usage_events/destructively_purge_all_and_reseed_existing_instances` | [api docs](http://apidocs.cloudfoundry.org/latest-release/service_usage_events/purge_and_reseed_service_usage_events.html) |

The purge endpoints are used to create initial events for all apps or service instances. The seeded events will all have the current timestamp, rather than the time the app was actually started. The purge endpoints will delete ALL existing events, so it is important to use them **only once** when you are starting to record usage events for the first time.

The purge endpoints are NOT intended to be called multiple times.  Calling these events multiple times may result in lost events.
