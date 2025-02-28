|         |                       |
| ------- | --------------------- |
| Author  | Ilia Kebets (Pryv SA) |
| Reviewers | Kaspar Schiess (Pryv SA) |
| Date    | 24.10.2018            |
| Version | 4                     |

Magic Invitations
{: .doc_title} 

Account creation tokens
{: .doc_subtitle} 

# Summary

As of today, a "magic non-expiring token" can be used to register on Pryv.IO. This token `enjoy` has been hard-coded into Registry.

This is a useful feature, but we have to let each customer decide to activate or change this token.

# Overview

Currently, the account creation API method is protected by the hard-coded weak token `enjoy`.

**The goal is for a Pryv.IO owner to grant and revoke user creation capability to multiple actors**. We need to provide our customers the possibility to handle a list of registration tokens and to modify this list if needed. 

If the Pryv.IO owner does not wish to restrict user creation, the list is not defined, hence accepting any value or no value at all. Therefore, we have backward compatibility with the current value set to `enjoy`.

Providing an empty list in the settings will prevent account creation.

# Design

We choose the minimal implementation for a quick release of this feature.

The changes will be in the done in the configuration as well as the invitationToken check.

The configuration will now have a `invitationTokens` field which stores an array of strings. By default this array is undefined, accepting any value or none at all.

The verification function will verify the token against the array of `invitationTokens` instead of the current hard-coded string.

# Out of scope

We will not implement a dynamic list configurable through the API since there does not seem to be such a need. The reboot of the register server takes <1 second, which is an affordable down time.

Implementing tokens that are usable a certain number of times or during a certain period of time is also not in the scope of this feature.

# Test / Quality Control

## Automated

Missing `invitationTokens` key in config  
- Succeeds if the `invitationToken` contains anything
- Succeeds if there is not `invitationToken` field  

Defined tokens list: `invitationTokens: ["abcde","abcdef"]`  
- Succeeds if the invitationToken matches one of them  
- Fails if the invitationToken does not match any token  
- Fails if the invitationToken is missing  

Empty tokens list: `invitationTokens:[]`  
- Fails for any invitationToken  
- Fails for missing invitationToken  

# Plan (time estimate)

We intend to modify the following artefacts:

- Registry:
  - Write tests (2h)
  - Add field to configuration with validation and default value (2h)
  - Refactor check (1h)