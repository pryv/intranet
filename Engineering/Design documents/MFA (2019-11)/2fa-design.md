|         |                       |
| ------- | --------------------- |
| Authors | Ilia Kebets |
| Reviewers | Thiébaud Modoux (v1,2), Simon Goumaz (v2) |
| Date    | November 6th 2019     |
| Version | 3                   |


# 2-Factor Authentication for Pryv.io

## Motivation

Provide increased security, validating a secret through a second channel in order to obtain a Pryv.io access token, necessary to make API calls.  

## 2FA activation

2FA should not be mandatory for all customers, therefore, it must first be activated before it is enforced in subsequent authentication processes:  

- After a successful sign in API call, a client can save a 2FA channel or secret in the Pryv.io [private profile resource](https://api.pryv.com/reference-full/#update-profile).
- After setting it up, the 2FA service will be able to check if 2FA is set for the concerned account by querying the private profile and looking for the `2fa` object.

**Example of 2FA profile properties:**

~~~yaml
profile:
  2fa:
    authenticator:
      secret: 'oi1n23ofaio12o3'
    sms:
      service: 'twilio'
      number: '+15017122661'
~~~

### Confirmation

There should be a step of confirmation that the 2FA is set correctly and available to the user before finalizing activation.  

## 2FA verification

### Sign in API call hook

Pryv.io implements 2-Factor Authentication (2FA) using an external micro-service. Sign in API calls\* are proxied by Pryv.io's NGINX to the 2FA micro-service where it takes over the authentication flow.

\* Currently HTTP POST on `/auth/login`

### Sign in call forwarding

The 2FA service forwards the Sign in API call to Pryv.io, *which is not proxied this time*.  
This can be done by either:  

- Calling the core container using its docker network hostname **if the 2FA service is running on the same machine**  
- Adding a secret in a HTTP header which tells NGINX to proxy the call to the core container instead of the 2FA service, **otherwise**. Example: [JWT](http://nginx.org/en/docs/http/ngx_http_auth_jwt_module.html)

### 2FA credentials retrieval

Having obtained a time-limited (personal) access token, it uses it to retrieve the 2nd factor secret (Authenticator) or channel (phone number, email address) from the Pryv.io private profile as covered in the activation chapter.  

### 2FA process execution & finish

The 2FA service launches its process and upon success, returns the time-limited access token to the client.

## Recovery

** /!\ TBD /!\ **

When losing the 2FA secret or channel, one must be able to get back access to it.

## Deliverables

### Documentation

We should provide documentation on how to:  

- Develop your own 2FA microservice
- Deploy it into your infrastructure

### 2FA microservices

We can develop and maintain 2FA microservices for some of the most often used ones such as Authenticator.


## Multiple Factor Authentication support

In case we support multiple FA systems and propose users to choose the one(s) they want, the logic must at some point in the flow, know which one to use.
After obtaining a valid personal token, it is possible to retrieve the activated MFA(s), therefore, we might have to bundle multiple FA services in one when a platform support more than 1.

# API routes

## Use case: secure channel (SMS or email)

We define 4 API routes, there are mainly 2 flows:
- Login, activate & confirm 2FA channel
- Login and verify 2FA code

Activate + confirm actions can be performed as many times as possible as long as you have obtained a ppToken (Pryv.io personal token).
We implement authentication of the verify route by a 2FA token whose validity is linked to the one of the used personal token. It allows to match the personal token's expiration and prevents brute forcing of this route in case a phone number (a somewhat public ID) is known.

### Login

**POST /login**

**Params:**
- username
- password

1. forwards to core
2.
  (2.1 if error, respond error)
  2.2 hold username/ppToken
  2.3 retrieve 2fa channel ID (phone number) from private profile
3. 
  3.1 if 2fa active - existence of entry `2fa:2faSystemId` means it is active
    - generate 2fa-token
    - save 2faToken->username/ppToken in RAM
    - send request to SMS API to send SMS
    - respond with:
      - 2faToken
      - some 2fa System identification?
    - jump to verify action

  3.2 if 2fa inactive
    - return return core's sign-in response

### Activate

**POST /activate**

**Params:**
- headers: 
    authorization: ppToken
- username? necessary if service hosted outside of pryv.io domain (username.pryv.io)
- phone number

1. check ppToken: GET username.pryv.io/access-info, if 4xx, forward response
2. saves number->username/ppToken
3. send request to SMS API to send SMS
4. respond with:
  - please confirm
  - status 201?
  - jump to /confirm

### Confirm

**POST /confirm**

**Params**
- header: 
    authorization: ppToken
- code
- username

same as verify between 2 and 4. (included)

5. save in private profile:

~~~yaml
profile:
  2fa:
    2faSystemId:
      number: XXX
~~~

6. return ppToken

### Verify

**POST /verify**

**Params:**
- header: 
    2fa-Authorization: 2faToken
- code
- username

1. retrieve ppToken/username matching to provided 2faToken
2. check: GET username.pryv.io/access-info
  - if it is invalid - invalidate 2faToken
3. retrieve number from RAM
4. send request to SMS API to verify code + number
5. return ppToken


## Concerns

### API calls audit

We wish that as much as possible of these calls are done through the [Pryv.io router](https://github.com/pryv/service-router). With the current implementation, we can audit all successful, failed and abandoned auth steps, *without distinction*, which are represented by API sign in calls. It would be better if we could filter between these cases, but might be sufficient as-is.

### Possible vulnerabilities

[Reference](https://shahmeeramir.com/4-methods-to-bypass-two-factor-authentication-2b0075d9eb5f)

Mainly, there are scenarii where web services bypass the default authentication flow, which in turn bypasses the 2FA. These are the following and should be taken in consideration when implementing them:

- Valid token after password reset - must not provide valid token in facilitation step  
- oAuth flow - must always use sign in call  
- rate limiting on 2FA code validation - don't forget to implement - done as 2faToken
- Limit in time 2FA code usage after its generation - limited by ppToken

## Out of scope

We only wish to enforce 2FA for the `/auth/login` call and not `accesses.create`.

## Side notes

### Private profile issues to fix

The documentation for the [private profile] resource is erronous:  the path is `/profile/private` instead `/profile/{id}`.  
When sending an empty body, api-server returns a status 500, which should be a 4xx.

## Annexes

### Authenticator flow

[guide](https://www.codementor.io/slavko/google-two-step-authentication-otp-generation-du1082vho)

- authenticator service generates secret
- authenticator provides secret to client over secure channel
- client authenticator sets up the secret and now generates OTP codes
- Now, the auth step does redirect to a authenticator server where the customer must provide a correct code
- if we stop here, we simply provide a token upon validating the 2nd step

### SMS (Twilio Verify)

[Verify API](https://www.twilio.com/docs/verify/api)

- client asks service to request code sending
- client receives code
- client sends code to service which verifies it
- when verified provide Pryv.io access token
















