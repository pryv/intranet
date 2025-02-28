|         |                       |
| ------- | --------------------- |
| Author  | Ilia Kebets (Pryv SA) |
| Reviewers | Pierre-Mikael Legris (Pryv SA) (v1) |
| Date    | August 26th 2019   |
| Version | 2                    |

Pryv.io Webhooks
{: .doc_title} 

Design document
{: .doc_subtitle} 

# Table of contents

- [Motivation](#motivation)  
- [Overview](#overview)
- [Design](#design)
  - [Core components structure](#core-components-structure)
  - [Parameters and state](#parameters-and-state)
    - [Uniqueness constraints](#uniqueness-constraints)
    - [Global parameters](#global-parameters)
  - [API routes](#api-routes)
  - [Security](#security)
- [Out of scope](#out-of-scope)
    - [Considered for future iterations](#considered-for-future-iterations)
- [Questions](#questions)
- [Tests and Quality control](#tests-and-quality-control)
- [Plan (time estimate)](#plan-time-estimate)

# Motivation

We need push notifications for Pryv.io data changes that don't require a persistent connection.

Currently, we can subscribe to notifications of **changes without its content** through a socket.IO connection.

Having webhooks would allow us the following:  
- Efficient resources usage: useful for unfrequent updates because no persistent connection is needed  

# Overview

A Webhook will be associated to an Access' permission set. It will be created and manageable using this Access' token.   

Once it is created and active, it will be executed up to a defined **maximum rate**, sending HTTP POST requests to the provided URL.  

The POST requests will contain an array of notifications similar to the ones sent by the current socket.IO notifications system:  

~~~~json
{
  "messages": [
    "eventsChanged",
    "streamsChanged",
    "accessesChanged"
  ],
  "meta": {
    "apiVersion": "1.4.8",
    "serverTime:": 1557927701.698,
    "serial": 20190810
  }
}
~~~~


When triggered, the Webhook will send all changes since its last execution.

Webhooks will turn off after a certain amount of **failed retries**, after which they will need to be restarted manually.

When the webhooks service is booted, it will send a "systemBoot" message to all active webhooks since we will might have lost notifications during down time.

Webhooks will also collect a certain amount of run execution analytics data.

# Design

## Functionality

### Accessibility

Only the access used to create the webhook (or a personal one) can be used to modify it. This is meant to separate the responsibilities between the webhooks management and services that will consume the data following a notification. 

Typically, a certain access will be used to setup one or multiple webhooks per user, where fresh data will be fetched using a different set of accesses.

#### Access deletion

Since the **access used to create the webhook** is separate from the one(s) used to consume the data, its deletion should not lead to the webhook's deletion.
Upon its deletion, the webhook will only be reactivable (needed if inactivated because of failed requests) using a personal token. Otherwise, a new access + webhook would need to be created.
For the same reason, we will not delete a webhook when its creating access expires.

### Failures

Webhooks will turn off after `maxRetries` of **failed retries**, after which they will need to be restarted manually.
This can be done using the [webhooks.update](https://api.pryv.com/reference-preview/#update-webhook) API method to change the webhook's `state`, which also resets the `currentRetries` field.
This route requires using the access that was used to create the webhook or a personal access. 

### Stats

We store run execution analytics data for each webhook in order to monitor their health such as [IFTTT's health view](https://platform.ifttt.com/services/pryv/health).

The current stats are:
- runCount   
- failCount   
- runs: last N (config `runsArraySize` number) entries of timestamp/code   
- lastRun: timestamp/code   
- currentRetries: current number of retries, up to `maxRetries`  

## Core components structure

As we have multiple independent `api-server` and `hfs-server` NodeJS processes which execute client commands, the `webhook-server` will be a separate NodeJS process that will receive notifications from the others through a message queue. We will reuse one of the currently available ones (NATS).    
The `webhook-server` will handle the notifications sending, handling parameters such as max sending rate enforcement and update the Webhooks storage objects.


## Parameters and state

[Webhook data structure preview](https://api.pryv.com/reference-preview/#webhook)

#### Uniqueness constraints

- AccessId, URL

### Global parameters

These global parameters will be defined in the webhook-server configuration file:  

- runsArraySize:
  - Description: The size of saved run arrays defined in the core configuration, defaults to 100
  - Format: Number
- minIntervalMs:
  - Description: The minimum period between Webhook calls, defaults to 5000 (5 seconds)
  - Format: Number
- maxRetries:
  - Description: The maximum number of retries permitted by a system before deactivating it, defaults to 5
  - Format: Number


## API routes

[API methods preview](https://api.pryv.com/reference-preview/#webhooks)  

## Security

### source authentication

- [Webhook signatures](https://stripe.com/docs/webhooks/signatures). The secret used to sign the payload is returned at webhook creation. To renew it, you must delete and create the webhook anew.  

### Limit URL scope

We need to prevent Webhooks from POST to components on internal networks such as the machines on the docker bridge or LAN.
I have currently no idea how to limit this, outside of forcing hostnames instead of IP addresses.
Since the account user cannot edit the payload outside of the messages set, we judge that allowing to discover hostnames in the private subnet through failures or successful runs, is a tolerable vulnerability.

# Out of scope

### Notification body

We might upgrade the notifications to sending the whole event. Currently it is out of scope, because sending events contents bypasses the router, **preventing the audit log flow**.

Uncoupled notified actor and token: the actor setting up the Webhook, who has the token, does not need to be the one receiving the notification

### destination authentication

As we only send changes notifications without the content, we do not include destination authentication such as:
- HTTP headers authentication  
- Destination authentication: generate a secret key upon Webhook creation that will be used to encrypt the Webhook payload upon sending and decrypt upon reception

# Questions


- Should we have notifications for webhooks changes?  

- How can dependent systems get updated of a system downtime?  
> The API needs a status API

- When booting a webhook-server, should we run all Webhooks? Yes, but it should not oveload the system.

# Tests and Quality control

## Automated

- [API routes](https://github.com/pryv/service-core/blob/feature/webhooks/components/api-server/test/webhooks.test.js)
- [Webhook business object](https://github.com/pryv/service-core/blob/feature/webhooks/components/business/test/acceptance/webhooks/Webhook.test.js)
- [Webhooks service](https://github.com/pryv/service-core/blob/feature/webhooks/components/webhooks/test/acceptance/service.test.js)

# Plan (time estimate - obsolete)

We intend to work a total of 12 days on the following artefacts:

- Core  
  - message queue  
    - figure out how it works (1d)  
    - reuse code if possible (1d)  
    - implement communication (1/2d)  
  - webhook-server  
    - write tests (2d)  
    - implement  
      - service runner (1d)  
      - execution loop (1d)  

  - storage  
    - webhooks objects (1d)    
  - api-server  
    - tests (2d)  
    - API routes (1d)  
    - API methods (1d)  
