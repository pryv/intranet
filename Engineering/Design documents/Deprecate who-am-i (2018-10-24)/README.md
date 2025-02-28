|         |                       |
| ------- | --------------------- |
| Author  | Ilia Kebets (Pryv SA) |
| Reviwer  | Kaspar Schiess (Pryv SA) |
| Date    | 24.10.2018            |
| Version | 3                     |

Deprecate `/who-am-i` route
{: .doc_title} 

Core API function presenting a security vulnerability
{: .doc_subtitle} 

# Summary

As of today, **core** sets a server-side cookie for the domain associated with the Pryv.IO platform upon successful login on the `auth.login` API method. This cookie allows to obtain the associated personal token when making a `HTTP GET` call on `https://${existing-username}.${domain}/who-am-i`.  
This creates a vulnerability that can be exploited by a malicious entity to steal personal tokens.

# Security vulnerability

A user with username `billy` has visited a web app which has done a successful sign in using the `auth.login` API method on his desktop or mobile browser.  
He is then invited by a malicious entity to open a certain page which he has crafted. The javascript running in this page sends a HTTP GET to `https://${existing-username}.${domain}` which returns the personal token of user `billy` on the associated Pryv.IO platform.  
The page sends the personal token to a server where it is stored to be exploited.

# Overview

This feature was meant to implement SSO, allowing a user to access multiple web apps in a browser without having to sign in for every single one of them.
The current design is broken as it presents the aforementioned security vulnerability. 
The goal is to ultimately remove this function from **core**.

As per our release manifest, we do not remove any functions of the API. Therefore, we will keep this API method simply deactivating it by default.
We will fully remove this implementation when we implement a fully working SSO for Pryv.IO

# Design

By default, this route returns an empty response with status 404. It is activated setting the configuration flag `deprecated.auth.ssoIsWhoamiActivated` to `true`.

When this route is activated, a warning is logged upon server startup.

# Test / Quality Control

To the current tests verifying the `GET /who-am-i` API method, we will add a test case for the default deactivated state and rename the current ones to indicate that the route has been activated.

# Plan (time estimate)

We intend to modify the following artefacts:

- Core:
  - Reorganize tests (30 min)
  - Write test for deactivated route (30 min)
  - Load `deprecated` settings in route `auths`, and add check for `ssoIsWhoamiActivated` in the route implementation (30 min)
  - Add `deprectated.auth.ssoIsWhoamiActivated` to config (30 min)
  - Add warning on server startup if `deprectated.auth.ssoIsWhoamiActivated` is set
  - Initiate file tracking changes for core v2 (1 h)
