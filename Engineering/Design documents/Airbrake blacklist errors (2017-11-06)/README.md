|              |                       |
| ------------ | --------------------- |
| Author       | Ilia Kebets (Pryv)    |
| Version      | 1 (6.11.2017)         |
| Distribution | Internally            |

## Issue

Since we have chosen to use the express handler for airbrake, all 4xx errors are sent to airbrake, which floods our errors inbox.

## Proposition

We have chosen to specify explicitly which errors we don't wish to have on Airbrake (blacklist approach).

### Motivation

We opted for this instead of a whitelist approach, so in case we forget to blacklist an error, it simply appears on Airbrake until we blacklist it, whereas using a whitelist approach, we might forget to whitelist an error which might lead to missing it on the airbrake reporting service.

### Implementation

To implement this, we extend the errors factory functions with an `options` object of the form `{dontNotifyAirbrake: true}` which will be merged with the `options` object of the `APIError(id, message, options)` constructor.

We will add a filter to the airbrake client to ignore these errors in a way similar to this:

```
airbrake.addFilter(function (notice) {
  if (notice.environment['err.dontNotifyAirbrake']) {
    return null;
  }
  return notice;
});
```
