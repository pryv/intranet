|         |                       |
| ------- | --------------------- |
| Author  | Pierre-Mikael Legris (Pryv SA) |
| Reviewers |  |
| Date    | May 31st 2019         |
| Version | 1                     |

Pryv.io API Services entry point
{: .doc_title} 

Design document
{: .doc_subtitle} 

# Motivation

### Problem

For many applications the first step is to Authenticate a user. Knowing the path to `https://access.{domain}/` is necessary. 

Most implementation even missuses this point and refer to `reg.{domain}` as it is the same machine. 

The worst effect is that applications have to **extract** the domain name from a passed parameter to initialize a connection. 

This leads to redundant, not consistent, not documented code in most of the apps that are usually code first with a hard coded domain, and fixed later for multi domain capabilities.

Most of our customers will need to disseminate the same information to their Apps.

### Proposal

A unified way for 3rd party services to access the necessary informations related to a Pryv.io implementation. 

# Overview

Finialize and document the usage of the `/service/infos` route

# Design

#### Informations

Needed informations for 3rd party & administration tools

##### Register API endpoint

- Used during the login process to convert an e-mail to a userId
- Used by administration tools to list users and manage invitation tokens

##### Access API endpoint

- Used during the App Auth request process

##### API endpoint

- When an App has the **username** it can establish what is the API endpoint to use

##### Event types

- The definition library of validated event-types by the system

##### UIs and UX related links and parameters

Used by UI(s) compatible with several Pryv.io platforms (About all Pryv demo tools)

- name
- home web page 
- support web page
- terms and conditons web page

#### Publishing and versioning

All Pryv.io services should respond to `/service/infos` route

- Register, Access, Core
- For synchronization among apps and services, a service info serial is published in the `meta` of Core responses as an incremental number.

#### A single way to expose a Pryv.io user API endpoint + token pair

With a single line parameter, services would be able to discover all the needed informations to operate. 


## Implementation requirements

#### Single line configuration.

`https://{token}@{user url endpoint}`

**Examples**: 

- Pryv.me : https://cjwbz6m0300871j405lqtfqsp@johndoe.pryv.me
- Specific port: https://cjwbz6m0300871j405lqtfqsp@johndoe.pryv.me:443/
- DNS less https://cjwbz6m0300871j405lqtfqsp@singlenode.com:443/johndoe/

**Motivation for this format**.

This will prevent tokens to be sent outisde of the *Headers* if clicked

##### Requirement:

`https://{user url endpoint}/service/infos` must respond 

##### Documentation to do:

Regexep in Javascript to extract the Authorization, Connection scheme and Endpoint in Javascript, Java

Example of usage for libraries.

#### /service/infos

Sample call result

```json
{
  "serial": "2019053101",
  "register": "https://reg.pryv.me:443",
  "access": "https://access.pryv.me/access",
  "api": "https://{username}.pryv.me/",
  "name": "Pryv Lab",
  "home": "https://pryv.com",
  "support": "http://pryv.com/helpdesk",
  "terms": "http://pryv.com/pryv-lab-terms-of-use/",
  "event-types": "http://api.pryv.com/event-types/flat.json"
}
```

Todo:

- To be implemented at register level.
- Core should load it at boot and cache it to respond to this call
  - Side effect, Service core should load event-types based on this
- Core should add a `service-infos-serial` in the meta of all its responses so app can update when needed. 
- Auth Demos, Apps & Libraries should take `https://{service}/service/infos` as parameter

##### Documentation

- API call should be added to the doc
- Good practice short guide should be edited