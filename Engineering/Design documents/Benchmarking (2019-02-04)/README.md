|         |                       |
| ------- | --------------------- |
| Author  | Ilia Kebets 		  |
| Date    | 04.02.2018            |
| Version | 1                     |


Pryv.io benchmarking
{: .doc_title} 

API calls performance
{: .doc_subtitle} 

# Motivation

We need a standard benchmarking framework in order to ensure that Pryv.io performance is consistent or better between releases.  
The results will be used to communicate numbers to our customers and to ensure that Pryv.io releases are up to the standards advertised in our specifications.

# Overview

We need to setup a benchmarking framework that will measure single API methods performance as well as close-to-real situations. The main goal is to compute Pryv.io's performance between releases in order to ensure that it is consistent or better at every iteration. 

In order to define close-to-real situations, we should request API usage patterns from our customers. Currently, we have pryv.me and Riva.

We will measure stats on each test and monitor performance client-side, but also include procedures to measure resource usage on the server side.

## Capacity VS overload

For each Pryv.io release, it is important to run tests using different load parameters, as the capacity will decrease when we reach an overload situation.

# Method

## Setup

- Before starting a run, we should measure the network capacity
- Tests should run for 30s, 5min, 10min, 30min  

## Results
We will compute the following parameters:  
- Fastest response  
- Slowest response  
- Mean response  
- Median response  
- 1st quartile?  
- 4th quartile?  
- Distribution?  

## API calls

- Tests should include various sized packages, defined in detail for each API method:   
  - events.get limits  
  - events.create payload size  
  - events.createWithAttachment file size
  - events.getAttachment file size
  - batch call size

For the following calls, we wish to choose the frequency of calls with their concurrency.  

### Register

- **users.create**

### Authentication

- **auth.login**
- auth.logout

### Events

- **events.get**
  - limit=1
  - limit=1k
  - limit=10k
- **events.create**
  - numerical: single value
  - complex: 10 fields
- **events.createWithAttachment**
  - file: 4kB
  - file: 500kB
  - file: 10MB
- **events.getAttachment**
- **events.update**
- events.delete

- **events.getOne**
- events.start
- events.stop
- events.addAttachment
- events.deleteAttachment

### Streams

- **streams.get**
- **streams.create**
- streams.update
- streams.delete

### Accesses

- accesses.get
- **accesses.create**
- accesses.update
- accesses.delete
- accesses.checkApp

### Utils

- accessInfo
- **batch**

### Followed Slices

- followedSlices.get
- followedSlices.create
- followedSlices.update
- followedSlices.delete

### Profile

- profile.getPublic
- profile.getApp
- profile.updateApp
- profile.get
- profile.update

### Account

- account.get
- account.update
- account.changePassword
- account.requestPasswordReset
- account.resetPassword

# Scenarii

We need to combine previous sets of calls together in run length, concurrency as well as frequency.

## Pryv.me

### Parameters
N users

### Per user
- events.get 50%
- events.create 25%

- streams.get 10%
- streams.create 5%

- events.createWithAttachment 5%
- events.getAttachment 5%

## Blood pressure

### Parameters
Frequency: how fast we launch a user's usage:  
  - 1s  
  - 5s  
  - 10s  
Users: how many users we launch  
  - 1  
  - 2  
  - 5  
  - 10  

#### Once
- user create
- login
- accesses.create
- batch streams.create
- events.create x 4
- events.get

#### Every 30seconds
- events.createWithAttachment
then  
- while: events.get limit=1

- events.getOne
- events.getAttachment
- events.get (limit=1) x 4 parallel
- events.get (limit=5)
- events.create
- events.update

## Home care

TODO

# Platform data

The Performance of the API depends on the load of the underlying storage. We define the following parameters influencing performance:
- Number of users  
- Number of streams/user  
- Number of events/user  
- Number of attachments/user  