|              |                       |
| ------------ | --------------------- |
| Author       | Thiébaud Modoux (Pryv)|
| Version      | 1 (20.11.2017)        |
| Distribution | Internally            |

# Introduce flag to turn off strictness with respect to protected fields for updates

## Description

In the last core release, we've introduced an new error that occurs when the client tries to update fields like 'id', 'modifiedBy' and the like. These fields are private fields and should not be externally updatable. 

Some components of the Pryv Ecosystem (Dashboard, Bridge-iFFFT, Bridge-Moves, Client Code) will not work anymore because of this.
We have thus decided to implement the following on top of the existing bugfix: 

* Default behaviour is strict checking and throwing an error. 

* For giving our clients and ourselves an easy migration path, we'll introduce the flag `ignoreProtectedFieldUpdates` that - if present and set to true - will ignore the field updates and log each operation that included ignored fields at the 'WARN' level. 

* The message could be as follows: 

  > Update to protected field(s) id, modifiedBy, ... was attempted on {Event,Access,...} with id XXX. Server has 'ignoreProtectedFieldUpdates' turned on: Fields are not updated, but no error is thrown. 

* Also, on server startup, I would suggest we log the following at the WARN level: 

  > Server configuration has 'ignoreProtectedFieldUpdates' set to true: This means updates to protected fields will be ignored and operations will succeed. We recommend turning this off, but please be aware of the implications for your code. 

## Impact in the code

### Configuration file

Configuration files will introduce a new setting 'ignoreProtectedFieldUpdates', a boolean which can be null, 'true' or 'false'.

### Update parameters validation function

Modifiying the way we handle forbidden updates will be easy since the same function is involved for all resources (Event, Streams, Accesses): [catchForbiddenUpdate](https://github.com/pryv/service-core/blob/master/components/api-server/src/methods/helpers/commonFunctions.js#L86).

Here we will check the 'ignoreProtectedFieldUpdates' setting in the config file and react accordingly: throw a forbidden error (current behavious) or call next() without error but print a warning log.

### Server start

Since ignoring forbidden updates errors is not a behavior that we want to encourage, we will look in the configuration file for the 'ignoreProtectedFieldUpdates' setting also at [server start](https://github.com/pryv/service-core/blob/master/components/api-server/src/server.js#L127) and print another warning message if activated.

### Testsuite

The described implementation must be tested against the following cases:
* set 'ignoreProtectedFieldUpdates' to null (or do not specify), perform a forbidden update, should throw forbidden error
* set 'ignoreProtectedFieldUpdates' to 'false', perform a forbidden update, should throw forbidden error
* set 'ignoreProtectedFieldUpdates' to 'true', perform a forbidden update, should not throw any error but should print a warn log
* warn log at server start can be manually tested
