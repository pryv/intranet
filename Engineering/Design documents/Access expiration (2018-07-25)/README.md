|         |                                  |
| ------- | -------------------------------- |
| Author  | Kaspar Schiess (kaspar@pryv.com) |
| Version | 1 (DRAFT, 20180725)              |





Expiring Accesses
{: .doc_title :}

Implementation Design
{: .doc_subtitle :}



# Summary

We extend access control to a Pryv.IO instance with the concept of 'expiry'. Accesses expire after a given number of seconds and cannot be used to access Pryv.IO afterwards. 

To implement this, we change the implementations of API methods 'accesses.get', 'accesses.create', 'accesses.update' and 'accesses.checkApp'. We extend the API documentation to reflect these changes. 

# Goals and Scope

We would like application accesses in Pryv.IO to optionally expire after a duration. This  can be specified upon creation or later on; once expired, an access cannot be used to authorise any action. 

We propose to introduce the relevant fields to store an expiry date in the 'access' entity. We also add API methods to manage that field and to be able to list expired accesses. 

We wont at this time change the 'app-web-auth2' application; functionality will be available to clients who implement their own version of app-web-auth only. 

Expiry is based on the server time of the relevant Pryv.IO core machine; precision will be limited to the precision of this server time. 

We don't implement a 'session' model here; expiry of an access - once set - remains fixed until it is changed; it doesn't prolong itself. 

No other changes are made at this time to the access control in Pryv.IO. 

# Use Cases

## #1 Access Expiry

1. The API Client configures an access to expire. 
2. Time passes. 
3. The Access Holder makes an API request. 
4. Pryv.IO denies the request.

Various concerns are not addressed in the description above; the use cases covers the primary goal of the implementation. 

# Design

## Adding Expiry to Accesses

We propose to add an optional field called `expires` to the documents in the users 'accesses' collection. This field encodes a date / time when the described access will stop working. 

**Who can expire accesses?** This field can only be changed if the authorising user can 'manage' the access; this is a prerequisite for any other update to an access, so no new permissions are introduced. The relevant code today stipulates that the authorising user uses a 'personal' access; this makes also sense for setting an expiry date on the access. 

**Behaviour** Expired accesses behave like regular accesses, with one exception: The list method ('accesses.get') will continue to return only accesses that work, excluding expired accesses. We will permit listing the expired accesses via another method (a variant of the existing). 

**Error Messages** When an operation on Pryv.IO is attempted using an access that has expired, the error message clearly reflects that the access has expired: "Access permission has expired.".

## API Extensions

**accesses.get** must NOT return accesses that have a configured expiry that is in the past, as measured by server time. 

We introduce the query parameter `includeExpired`. When set to `true`, the list returned by this method will include accesses that are in the past. The parameter defaults to `false`.

**accesses.create** To create an access that expires, one can specify `expiresAfter` as attribute of the access. This field contains the number of seconds after which the access expires. 

If successful, the operation will return an access that includes the `expires` field in epoch timestamp format, defining the epoch time when the access expires. 

We structure the API call this way to avoid sketchiness associated with timezones and interpretation of absolute dates. The server performs the computation of when exactly the access expires.

If `expiresAfter` is set to zero (0) and the call is otherwise successful, the return value will be an access that has already expired. Negative values are not allowed in this field and will return an error message. 

**accesses.update** Access 'expiry' can be set and cleared using an update. To set a new expiration date for the access, specify the `expiresAfter` parameter in the call. 

To remove expiry from an access (turn it into an access that doesn't expire), one needs to include the field `expires` in the update, specifying `null` as value. 

**accesses.checkApp** An expired access is considered a 'mismatch'; meaning that it is still returned, but in the field `mismatchingAccess`. 

## Code Locations

Wherever possible, we will aim to centralise important checks in the access code  to avoid forgetting checks down the road. For example, the code in 'MethodContext.js:147' (retrieveAccess) will probably end up being the only place we 'findOne' an access. 

# Tests

## Automatic

* [x] Create an access with an expiry
* [x] Update an access to expire
* [x] Remove expiry via Update
* [x] `expiresAfter` set to 0 produces an expired access on create
* [x] `expiresAfter` set to 0 produces an expired access on update
* [x] Expired accesses can be updated. 
* [x] List doesn't include expired accesses. 
* [x] List can include expired accesses when given `includeExpired: true`. 
* [x] API accesses using expired access fail. 
* [x] Check error message when it fails.
* [x] An access cannot update itself.
* [x] checkAccess should not consider an expired access to be a match.



## Acceptance / Manual

Let's test the new expiring access feature in Pryv.IO. First, we'll need a token that can access the system and create - in turn - other accesses. 

To do a local test, please check out the repository 'git@github.com:pryv/ops-local_build_test.git' as a sibling to 'service-register' and 'service-core' git repositories. You should change into it and run 'bin/launch'. Normally, you see a few pages scrolling by, but no errors - you should now have a valid rec.la installation. 

I navigate to [app-web-access](http://pryv.github.io/app-web-access/?pryv-reg=reg.rec.la) and  get a master token for my pryv.me account (the master token is not really neccessary; but it makes my test here simpler, so I chose that). The token I get is 

| **Username**     | kschiess                  |
| ---------------- | ------------------------- |
| **Access token** | cjk3rwz0k000334jnq9ooqg3m |
| **Domain**       | rec.la                    |

Starting from that, let's create a normal child access using the example from the API documentation. 

```bash
$ curl -kv -X POST \
	-H "Authorization: cjk3rwz0k000334jnq9ooqg3m" \
	-H "Content-Type: application/json" \
	-d '{"name":"For colleagues","permissions":[{"streamId":"work","level":"read"}]}' \
	https://kschiess.rec.la/accesses | jq '.'
```

(My rec.la certificate is currently broken. I'll add the '-k' to all the commands here, maybe you don't have to.)

Result: 

```json
{
  "meta": {
    "apiVersion": "1.3.19-292-g1ba0067",
    "serverTime": 1532683280.202
  },
  "access": {
    "name": "For colleagues",
    "permissions": [
      {
        "streamId": "work",
        "level": "read"
      }
    ],
    "type": "shared",
    "token": "cjk3s1vt3000033jnc57j6dr1",
    "created": 1532683280.199,
    "createdBy": "cjk3rwz0k000434jnz4udfwre",
    "modified": 1532683280.199,
    "modifiedBy": "cjk3rwz0k000434jnz4udfwre",
    "id": "cjk3s1vt4000133jnfytijnq5"
  }
}
```

And then, let's list the accesses we can manage to see if the new access shows up in the list: 

```shell
curl -kv -H "Authorization: cjk3rwz0k000334jnq9ooqg3m" \
	https://kschiess.rec.la/accesses | jq '.'
```

```json
{
  "meta": {
    "apiVersion": "1.3.19-292-g1ba0067",
    "serverTime": 1532683339.81
  },
  "accesses": [
    {
      "name": "For colleagues",
      "permissions": [
        {
          "streamId": "work",
          "level": "read"
        }
      ],
      "type": "shared",
      "token": "cjk3s1vt3000033jnc57j6dr1",
      "created": 1532683280.199,
      "createdBy": "cjk3rwz0k000434jnz4udfwre",
      "modified": 1532683280.199,
      "modifiedBy": "cjk3rwz0k000434jnz4udfwre",
      "id": "cjk3s1vt4000133jnfytijnq5"
    }
  ]
}
```

Now that we've got an access to manage, let's create a second access that will expire in about a minute's time. 

```shell
curl -kv -X POST \
	-H "Authorization: cjk3rwz0k000334jnq9ooqg3m" \
	-H "Content-Type: application/json" \
	-d '{"name":"For Elise","expireAfter":60,"permissions":[{"streamId":"work","level":"read"}]}' \
	https://kschiess.rec.la/accesses | jq '.'
```

```json
{
  "meta": {
    "apiVersion": "1.3.19-292-g1ba0067",
    "serverTime": 1532683388.923
  },
  "access": {
    "name": "For Elise",
    "permissions": [
      {
        "streamId": "work",
        "level": "read"
      }
    ],
    "type": "shared",
    "token": "cjk3s47p7000534jnxzhsd85c",
    "expires": 1532683448.923,
    "created": 1532683388.923,
    "createdBy": "cjk3rwz0k000434jnz4udfwre",
    "modified": 1532683388.923,
    "modifiedBy": "cjk3rwz0k000434jnz4udfwre",
    "id": "cjk3s47p7000634jnvojr14kk"
  }
}
```

Looks like we've got ourselves our first access that expires; decoding the timestamp, it shows that it will expire on 2018-07-27 11:24:08 +0200, and server time of the request was 2018-07-27 11:23:08 +0200. This is exactly one minute, as expected. 

Let's use this access for something (while we can): 

```shell
curl -kv -H "Authorization: cjk3s47p7000534jnxzhsd85c" \
	https://kschiess.rec.la/events | jq '.'
```

Results in 

```json
{
  "events": [],
  "meta": {
    "apiVersion": "1.3.19-292-g1ba0067",
    "serverTime": 1532683641.306
  }
}
```

Once the access expires, the result from the above curl becomes: 

```json
{
  "error": {
    "id": "forbidden",
    "message": "Access has expired."
  },
  "meta": {
    "apiVersion": "1.3.19-292-g1ba0067",
    "serverTime": 1532683526.119
  }
}
```

That's too bad. Let's prolong the access by one minute exactly: 

```shell
curl -kv -X PUT \
	-H "Authorization: cjk3rwz0k000334jnq9ooqg3m" \
	-H "Content-Type: application/json" \
	-d '{"expireAfter":60}' \
	https://kschiess.rec.la/accesses/cjk3s47p7000634jnvojr14kk | jq '.'
```

Result is: 

```json
{
  "meta": {
    "apiVersion": "1.3.19-292-g1ba0067",
    "serverTime": 1532683663.21
  },
  "access": {
    "name": "For Elise",
    "permissions": [
      {
        "streamId": "work",
        "level": "read"
      }
    ],
    "type": "shared",
    "token": "cjk3s47p7000534jnxzhsd85c",
    "expires": 1532683723.208,
    "created": 1532683388.923,
    "createdBy": "cjk3rwz0k000434jnz4udfwre",
    "modified": 1532683663.208,
    "modifiedBy": "cjk3rwz0k000434jnz4udfwre",
    "lastUsed": 1532683641.304,
    "id": "cjk3s47p7000634jnvojr14kk"
  }
}
```

If we want to expire the access immediately, we can PUT an expireAfter of 0: 

```shell
curl -kv -X PUT \
	-H "Authorization: cjk3rwz0k000334jnq9ooqg3m" \
	-H "Content-Type: application/json" \
	-d '{"expireAfter":0}' \
	https://kschiess.rec.la/accesses/cjk3s47p7000634jnvojr14kk | jq '.'
```

To remove the expiry from the access, we'll PUT this: 

```shell
curl -kv -X PUT \
	-H "Authorization: cjk3rwz0k000334jnq9ooqg3m" \
	-H "Content-Type: application/json" \
	-d '{"expires":null}' \
	https://kschiess.rec.la/accesses/cjk3s47p7000634jnvojr14kk | jq '.'
```

```json
{
  "meta": {
    "apiVersion": "1.3.19-292-g1ba0067",
    "serverTime": 1532683778.764
  },
  "access": {
    "name": "For Elise",
    "permissions": [
      {
        "streamId": "work",
        "level": "read"
      }
    ],
    "type": "shared",
    "token": "cjk3s47p7000534jnxzhsd85c",
    "expires": null,
    "created": 1532683388.923,
    "createdBy": "cjk3rwz0k000434jnz4udfwre",
    "modified": 1532683778.763,
    "modifiedBy": "cjk3rwz0k000434jnz4udfwre",
    "lastUsed": 1532683641.304,
    "id": "cjk3s47p7000634jnvojr14kk"
  }
}
```

Now we can use it to list events again: 

```shell
curl -kv -H "Authorization: cjk3s47p7000534jnxzhsd85c" \
	https://kschiess.rec.la/events | jq '.'
```

```json
{
  "events": [],
  "meta": {
    "apiVersion": "1.3.19-292-g1ba0067",
    "serverTime": 1532683804.443
  }
}
```

This sums up what can be done using expiring accesses. Below we'll verify a couple of special situations that are less useful for this tutorial but need to be tested as well. 

**Can an access modify itself and prolong its own expiry?** Normally, no, but let's try this out: 

```shell
curl -kv -X PUT \
	-H "Authorization: cjk3s47p7000534jnxzhsd85c" \
	-H "Content-Type: application/json" \
	-d '{"expireAfter":10}' \
	https://kschiess.rec.la/accesses/cjk3s47p7000634jnvojr14kk | jq '.'
```

```json
{
  "error": {
    "id": "forbidden",
    "message": "You cannot access this resource using a shared access token."
  },
  "meta": {
    "apiVersion": "1.3.19-292-g1ba0067",
    "serverTime": 1532683945.804
  }
}
```

So no, you cannot. 

**Expired accesses should disappear from the access list.** Let's see if this is true: 

```shell
curl -kv -X PUT \
	-H "Authorization: cjk3rwz0k000334jnq9ooqg3m" \
	-H "Content-Type: application/json" \
	-d '{"expireAfter":0}' \
	https://kschiess.rec.la/accesses/cjk3s47p7000634jnvojr14kk | jq '.'

curl -kv -H "Authorization: cjk3rwz0k000334jnq9ooqg3m" \
	https://kschiess.rec.la/accesses | jq '.'
```

Yes, it is gone here. Let's see if we can get it back: 

```shell
curl -kv -H "Authorization: cjk3rwz0k000334jnq9ooqg3m" \
	'https://kschiess.rec.la/accesses?includeExpired=true' | jq '.'
```

Yes, back it is. 