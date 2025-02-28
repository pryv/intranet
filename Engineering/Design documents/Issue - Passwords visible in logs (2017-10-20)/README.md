## Issue

Passwords in clear and hashed appear in the logs when calling `/system/create-user` and `/auth/login` routes and errors occus.
Hereafter, are exposed the cases and their fixes. The general idea is to replace the `password`/`passwordHash` value by `########` (or equivalent) in `req.body` after it has been copied in the `params` object.  
Measures to avoid printing credentials in the case when field keys are misspelled (such as `{passwordd: 'confidential'}`) were dropped because these shouldn't happen in production. In addition, whitelisting fields will hinder the capacity to detect genuine errors in field names during development phases.

## create-user

When an error occurs, the password hash is leaked in the following form:

```
info: [api::routes] item-already-exists error (400): An user with username "mr-dupotager2" already exists location=/system/create-user/, method=POST, username=mr-dupotager2, passwordHash=1f4Dj6s47cCdwR4tzmnNfelSkpPuJ9w318hsvBxhFpNSEBVQT5rFq, email=dupotager2@test.com, language=fr, username=mr-dupotager2, innerError=E11000 duplicate key error collection: pryv-node.users index: username_1 dup key: { : "mr-dupotager2" }
```

### 1. Invalid authorization token

The logger explicitly prints the req.body [here](https://github.com/pryv/service-core/blob/master/components/api-server/src/routes/system.js#L36), we can remove or hide the hash here.

### 2. Error in the creation process (User already exists, invalid parameters, concurrent requests)

A solution would be to clear req.body.passwordHash [in routes/system](https://github.com/pryv/service-core/blob/master/components/api-server/src/routes/system.js#L51), since we access it through another variable later (we only need req for error management which involves logging where we don't want the password hash to be).

Proposal:

``` 
function createUser(req, res, next) {
    let params = _.extend({}, req.body);
    req.body.passwordHash = '###########';
    systemAPI.call('system.createUser', {}, params, methodCallback(res, next, 201));
  }
```

The error is thrown by the DB, an API error is provided to the errors middleware which [accesses the req.body](https://github.com/pryv/service-core/blob/master/components/errors/src/errorHandling.js#L23)
It is not possible to clear the passwordHash after the error has been thrown because we have no access to req in methods/system [at this point](https://github.com/pryv/service-core/blob/master/components/api-server/src/methods/system.js#L45)

### cURLs

Create user on remote

```
curl -k -i -X POST -H 'Content-Type: application/json' -H 'authorization: ZSBt6e5xash' -d '{"username": "mr-dupotager2","passwordHash": "this-is-my-leaked-password-hash", "email": "dupotager2@test.com","language": "fr"}' "https://stcore-azure-nl-01.pryv.net/register/create-user/"
```

Create user on local

```
curl -i -X POST -H 'Content-Type: application/json' -H 'authorization: OVERRIDE ME' -d '{"yolo":"yala","username": "mr-dupotager2","passwordHash": "1f4Dj6s47cCdwR4tzmnNfelSkpPuJ9w318hsvBxhFpNSEBVQT5rFq", "email": "dupotager2@test.com","language": "fr"}' "http://127.0.0.1:3000/system/create-user/"
```

## login

If an error happens when executing the login, the string provided in the `password` field is printed in clear in the logs:

```
info: [api::routes] invalid-credentials error (401): The app id ("appId") is either missing or not trusted. location=/userzero/auth/login, method=POST, username=userzero, password=blablabla, appId=pryv-test
```

### Any case (not going into details)

We should hide the password in req.body in `routes` as the normal execution doesn't access the password through req.body, it simply leaks when the request dump is logged.
I suggest replacing the password with something like "###########" instead of simply erasing it to see in the logs that a password was provided for easier debugging.

Proposal:

```
expressApp.post(Paths.Auth + '/login', function (req, res, next) {
    var params = {
      username: req.body.username,
      password: req.body.password,
      appId: req.body.appId,
      origin: req.headers.origin || ''
    };
    req.body.password = '############'
    api.call('auth.login', req.context, params, function (err, result) {
      if (err) { return next(err); }
      setSSOCookie({ username: req.context.username, token: result.token }, res);
      result.writeToHttpResponse(res, 200);
    });
  });
```

### cURLs

Login on local:

```
curl -i -H "Content-Type: application/json" -H "Origin: https://sw.rec.la" -X POST -d '{"username":"userzero","password":"t3st-Z3r0","appId":"pryv-test"}' "http://userzero.rec.la:3000/auth/login"
```

Login on remote:

```
curl -i -H "Content-Type: application/json" -H "Origin: https://sw.pryv.li" -X POST -d '{"username":"testuser","password":"testuser","appId":"pryv-test"}' "https://testuser.pryv.li/auth/login"
```

