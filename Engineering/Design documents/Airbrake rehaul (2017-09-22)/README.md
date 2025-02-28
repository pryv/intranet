|         |                 |
| ------- | --------------- |
| Author  | Thiébaud Modoux |
| Version | 2 (22.9.2017)   |

# Configure Airbrake for better context in error reports

This document describes our needs in improving Airbrake reports so that errors are shown with better context. It proposes some solutions and presents the expected changes in the code.

## Needs:

In our Airbrake reports, we would like to see each error with the following contextual details:

* A valid URL and hostname (instead of a docker container id)
* A more precise error severity
* A stacktrace of the error containing the faulty request

## Solutions:

For all the code parts where we use Express, we will just rely on the [Airbrake-Express integration](https://github.com/airbrake/node-airbrake#express-integration) for reporting errors.

Where this Express integration can not be used (raw calls), we will augment each error with contextual information before explicitly call the [Airbrake notify function](https://github.com/airbrake/node-airbrake#manual-error-delivery).

According to the [Airbrake documentation](https://github.com/airbrake/node-airbrake#adding-context-to-errors), here is a list of contextual information which make sense:

* URL/hostname: err.url
* Error type: err.class ('Error' if not set)
* Error severity: err.severity
* Stacktrace: err.stack

## Changes in code:

### Service-core

Cores currently configure Airbrake through a Logging component:

* Activate the default exceptions handler: [here](https://github.com/pryv/service-core/blob/master/components/utils/src/logging.js#L50)
* Explicitly notify custom errors: [here](https://github.com/pryv/service-core/blob/master/components/utils/src/logging.js#L101)

Activating the default exceptions handler is fine but instead of explicitly notify errors (with poor context),
we will rely on the Airbrake-Express integration.

To do so, when initializing the [express app](https://github.com/pryv/service-core/blob/master/components/api-server/src/expressApp.js#L28),
we will add the following code: 

```
var airbrake = require('airbrake').createClient("project ID", "api key");
app.use(airbrake.expressHandler());
```

### Service-register

Registers currently rely on the default Airbrake exceptions handler, activated at [server setup](https://github.com/pryv/service-register/blob/master/source/server.js#L61).

Similarly to cores, we will additionally activate the Airbrake-Express integration when initializing the [express app](https://github.com/pryv/service-register/blob/master/source/app.js#L28), in order to report all web-based errors.

For errors unrelated to Express (dns component for example), we will use an error middleware to catch these errors,
augment them with more context and explicitly call the Aibrake notify function.

Such [error middleware](https://github.com/pryv/service-register/blob/master/source/middleware/app-errors.js) is already existing in registers (but not used) so it would be easily improved and activated.