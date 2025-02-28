|         |                       |
| ------- | --------------------- |
| Author  | Thiébaud Modoux (Pryv SA) |
| Reviewers |  |
| Date    | June 12th 2019            |
| Version | 1                     |

Pryv.io Audit API
{: .doc_title} 

Design document
{: .doc_subtitle} 

# Table of contents

- [Motivation](#motivation)  
- [Overview](#overview)
- [Design](#design)
- [Out of scope / Additional features](#out-of-scope-additional-features)
- [Questions](#questions)
- [Tests and Quality control](#tests-and-quality-control)

# Motivation

Now we have implemented a first version of a Pryv.io audit router ([design](https://int.pryv.com/PryvIO/InternalDesignJournal/20190304-pryv-io-audit), [code](https://github.com/pryv/service-router)), which logs API calls details to syslog, we would like to provide a friendly way of querying and searching in these logs.

# Overview

As described within the service-router [README](https://github.com/pryv/service-router/blob/master/README.md), Pryv.io audit consists of logging details about each API requests and responses to the syslog (see [here](https://github.com/pryv/service-router/blob/master/README.md#log-format) for the log format and [here](https://api.pryv.com/assets/docs/20190508-pryv.io-audit-v3.pdf) for the log files organization).

These logs will end up accumulating in log file(s), from which we would like to be able to extract specific records, according to some search criteria (e.g. show logs of every calls performed using specific Pryv token).

This is why we are going to implement a query API for the Pryv.io audit log.

# Design

The audit (REST) API will consist of a HTTP server, serving an Express application with few routes that allow to GET specific log records from the syslog file(s) according to several search criteria, provided as query parameters.

Since for now the audit logs are piped to file(s) by syslog, we chose to rely on a 'grep' strategy as first solution for extracting specific logs. In others words, we will implement a Search interface, searching plain-text data sets for lines that match some regular expressions, using grep commands (or a similar tool).

The current design should allow to easily switch from the 'grep' strategy, to database querying if one day we allow the Pryv.io audit to pipe logs to a database.

## API Endpoints

The Express application will serve thwo main endpoints. Both will enforce to provide an 'authorization token' as **Authorization** header, through an Authorization (Express) middleware.

### GET {username}.{domain}/audit/logs

Retrieves log entries from syslog logs for a specific user and current access id.

If the access id to look for is not specified as query parameter, we will assume that the token provided in the Authorization header needs to audit itself.

In this case, the Authorization middleware just has to perform a GET /access-info API call against service-core, in order to check the validity of the token and retrieve its corresponding access id (within Pryv-access-id header).

Finally, the scope of the search results will then be limited to audit entries that contain this exact access id.

In the other hand, if a specific access id is provided as query parameter, it will retrieve log entries from syslog logs for a specific user and the given access id.

In this case, the Authorization middleware will first perform a GET /accesses API call against service-core, forwarding the same Authorization header.

It has the effect of checking the validity of the provided authorization token, and to retrieve the list of all accessible Accesses.

If this list contains an Access with access id matching the one provided during the audit call, it means that the token provided in the Authorization header is allowed to audit the access id provided in request path.

This call fits when a token needs to audit another 'sub-token'. If an auditor needs to audit several 'sub-tokens', he first needs to retrieve the list of all accessible tokens and repeat call 1) for each of the corresponding access ids.

## Query parameters

When calling this endpoint, we can set several search criteria by providing them as query parameters. Here is a list of these search properties:

- Access id
- Date: fromTime, toTime
- Http code: range or single
- Details
	- forwarded for: client ip
	- action: HTTP verb, path
	- error id

All the query parameters will be passed to the Search interface, so that it can build the search query before executing it.

## Search interface

The Search interface will consist mainly of a Filter class, which will be instanciated with a map of property:value provided by the Express application after extracting the query parameters.

For each of the properties listed above, the Search interface will have a corresponding function, which allow to augment the Filter with an additional search property:value. When augmenting the Filter, we typically attach to it a new search expression/command, which is specific to each search property.

Finally, the Filter will have a function that constructs the final query from all the registred search properties. In the context of our 'grep' strategy, this final query consists of the concatenation of several 'grep' commands, each one using a specific dynamic regular expression.

It should then be easy to change the implementation of the Filter so that it constructs a database query as final query instead of a concatenation of 'grep' commands, in case we plan to store the audit logs in a database.

Below, we give some examples of search commands (using grep) for each search property.

### Date

Example: `awk -F - '"Jun 12 14:10:19" < $1 && $1 <= "Jun 12 14:10:20"'`

Timestamps for fromTime and/or toTime can be specified as query params.

They will need to be converted to date to match the logging format.
Then it will be possible to filter the correct time range, accurate to the nearest second.

If fromTime, respectively toTime, is omitted, it will be set to 0, respectively current time.

### HTTP code

Example: `grep '400:'`

The exact HTTP code (400) to look for can be specified as query params.

We will also allow to ask for a range of HTTP code by only providing the first digit (4) of the HTTP codes to look for, so the others two digits will act as wildcard.

### Forwarded for

Example: `grep '"forwarded_for":"172.18.0.7"'`

The exact originating (client) IP (172.18.0.7) to look for can be specified as query params.

### Action

Example: `grep '"action":"GET /events"'`

The HTTP verb (GET) and the request path of the action can be specified as query params. Both HTTP verb or request path can be omitted so that it acts as wildcard.

### Error id

Example: `grep '"error_id":"invalid-operation"'`

The error id (invalid-operation) can simply be specified as query params.
It can be more precise than searching by HTTP code.

## API results

Audit call results should mimic the GET events API calls against service-core.
This means that the filtered log records need to be converted to a structure similar to Events (with type and content)
and accumulated in a result array, possibly streamed.

This will be the role of a Result class, which will take the filtered log records as input,
extract the main details from each of them using a regexp and construct a corresponding JSON that will be the Event content.

We can imagine adding the possibility to return the raw log records as text (if the parsing fails for example).

## Configuration

- Log location (folder/file(s) where to search for logs). We expect to have at least one log folder per user, as configured by [syslog configuration](https://api.pryv.com/assets/docs/20190508-pryv.io-audit-v3.pdf).
- Service-core endpoint

# Out of scope / Additional features

- Audit the audit calls. This will require to add the audit api-server(s) as new BackendType in service-router. 
- Retrieve requests and responses grouped by request id
- Expert search directly piped to grep
- Limit the results with a limit query parameter (using grep -m)
- Additional search properties:
  - Backend: useful only if we add more backend types
  - Programname: may be set to pryvio_core? not so useful otherwise
  - Response/error message: not so useful
  - Details
  	- auth_hash
  	- request id
  	- username: implies that we allow cross-users audit
  	- query: search in query string, format key1=value,key2=value, check for presence only (omit value), look for multiple params (any order)
  	- forwardedFor: range of ip instead of exact ip

# Questions

- How to handle log rotation? Grep VS compressed logs?

# Tests and Quality control

Test suite will consist mainly of acceptance tests.

1) Configuration
- Successully loads the log location
- Successfully targets core endpoint
- Returns adapted error if the log location is invalid
- Returns adapted error if the core endpoint is unreachable

2) Authorization: Mock the service-core endpoint
- A valid token can successfully audit a subtoken
- A valid token can successfully audit itself
- Returns adapted error if Authorization header is not specified
- Returns adapted error if provided token is not authorized to audit provided access id
- Returns adapted error if provided token is not authorized to audit itself (expired, invalid)

1) API: Spawn the Express app, configure it to target dummy log file(s) and compare the APIResults:
- Main endpoints are reachable
- Returns adapted error if targeted endpoint does not exist
- Returns adapted error if query param format is invalid (for each available query param)
- Successful GET by Date providing fromTime
- Successful GET by Date providing toTime
- Successful GET by Date providing fromTime and toTime
- Successful GET by HTTP code providing exact code
- Successful GET by HTTP code providing range of codes
- Successful GET by Forwarded For providing IP
- Successful GET by Action providing HTTP verb and request path
- Successful GET by Action providing HTTP verb
- Successful GET by Action providing request path
- Successful GET by Error id
