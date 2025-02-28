# Review and adaptation of mailing solution for Pryv installations
This document describes the mailing solution adopted by all the Pryv installations, points out the weaknesses of the current implementation and proposes an alternative implementation.

This is the version 2 of this document. Document owner is Thiébaud Modoux/Pryv.

## Current situation:

We currently use **[Mandrill](http://www.mandrill.com/)**, a transactional email API, to send emails to Pryv users who just registered or asked for lost password.
Moreover, Mandrill allows to define some email templates.

When a new user is created or when a password reset is requested, Pryv core send HTTP requests to Mandrill API to order the sending of an email.

## Problems:

Having such code in our cores implies that we enforce our customers who install a Pryv instance to use this service (inscription confirmation and lost password emails).
Furthermore, we limit them to the usage of Mandrill.

Some customers may want to deactivate this service; for example, customers who will administrate users outside Pryv.

Some customers may want to use their own solution for this service, it can be Mandrill or any other similar solution.

Some customers may want to have a ready-to-use solution but to be able to customize this service, by providing email templates for example.

In conclusion, we currently can not satisfy these various customer needs with our current approach.

## Implementation proposition:

### Overview
According to the expected customer needs listed above, we have to find a solution that makes Pryv core have the following properties:

  * Include a basic built-in mailing service
  * Allow the customization of the built-in mailing service
  * Allow to deactivate the built-in mailing service
  * Allow to plug in external mailing modules
  
### Mailing module
According to the previous points, we could have a built-in mailing module that defines the following methods:
  * sendPasswordResetMail: load template, generate and format the email and call "sendMail"
  * sendWelcomeMail: load template, generate and format the email and call "sendMail"
  * sendMail: actually send an email, define the mailing strategy

Moreover, this module should read a configuration file (defined at deployment) to activate/deactivate itself (boolean variable) or to load custom templates (path to html/css files, which will be provided by our customers).

Finally, this module should be overridable by a custom mailing module implementing the required methods listed above.
From an external point of view, the mailing module should work indifferently whether built-in or custom.

### Mailing service
Now we have described the structure of the mailing module, we have to define how we will actually send emails.
Custom mailing modules will use their own mailing service so we will only plan the built-in mailing service here.

We have to define the built-in mailing strategy by implementing the "sendMail" function of the mailing module.
A basic solution would be to have the core directly sending the mail using the SMTP server of the hosting provider (not sure about this, still need some research).

## Impact on current implementation
In this section, we will inspect the current implementation and think about the possible changes to reach the proposed implementation. Please take a look at the code reference at the beginning of each subsection while going through the explanations for a better comprehension.

### Configuration
Code reference: [service-core/build/core/config/core.json](https://github.com/pryv/service-core/blob/master/build/core/config/core.json#L38)

We currently have a configuration file for core that defines some mailing properties.

Among them, "url", "key" and "sendMessagePath" should be removed since it will be internal properties of the mailing module.

The other properties "welcomeTemplate" and "resetPasswordTemplate" can be kept and will be used by the mailing module to load custom email templates.

### Password reset email
Code reference: [service-core/components/api-server/src/methods/accounts.js](https://github.com/pryv/service-core/blob/master/components/api-server/src/methods/account.js#L76)

We currently have a "requestPasswordReset" route to request a password reset, which use a "sendPasswordResetMail" function to send the reset email.
Note that this email have to provide a reset token, generated by the "generatePasswordResetRequest" function.

We should replace the sending function with the "sendPasswordResetMail" function of our mailing module, which will take the generated reset token as parameter.

We also have to check here if the mailing module is activated. If not, we will not send the email but just return the reset token, which can then be used to reset the password through the [corresponding route](https://github.com/pryv/service-core/blob/master/components/api-server/src/methods/account.js#L130), for customers that want to handle user administration from external side.

### Welcome email
Code reference: [service-core/components/api-server/src/methods/system.js](https://github.com/pryv/service-core/blob/master/components/api-server/src/methods/system.js#L27)

Currently, our user creation route is sending a welcome email at the end of the user creation process by calling the "sendWelcomeMail" function.

This function should just be replaced with the "sendWelcomeMail" of our mailing module.

Here again, we have to check if the mailing module is activated. If not, nothing special to do, we will not send the email.

## Short term solution
We also propose a short term solution for pilots that need to turn off (partially or completely) mail sending.

This will be done by introducing new settings to 'services.email', in the configuration file: 
```
    "email": {
      // old 
      "url": "https://mandrillapp.com",
      "key": "OVERRIDE ME",
      "sendMessagePath": "/api/1.0/messages/send-template.json",
      "welcomeTemplate": "welcome-email",
      "resetPasswordTemplate": "reset-password"
      // new
      "enabled": {
      	"welcome": true, 
        "resetPassword": false, 
      }
    }
```

Alternatively, we can allow "enabled": false as well, disabling all the emails. 