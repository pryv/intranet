### initContext

`req.context` loaded in `api-server/src/routes/root.js#29`:

```javascript
expressApp.all(Paths.UserRoot + '/*', initContextMiddleware);
```

1. retrieves the auth token from all the possible places - this is done in multiple places (bad)
   1. `initContext.getAuth()`
   2. `MethodContext.parseAuth()`
   3. 
2. retrieves user
3. instanciates `MethodContext` object



### loadAccess

it is a simple function (not express middleware) that runs `context.retrieveExpandedAccess()`.



### Access methods

They are scattered between `model/MethodContext` and `model/accessLogic`.



