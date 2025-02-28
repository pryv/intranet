|         |                       |
| ------- | --------------------- |
| Author  | Ilia Kebets |
| Reviewer  | Kaspar Schiess (V1,2), Guillaume Bassand (V1) |
| Date    | 07.11.2018           |
| Version | 3                    |

Email address (and username)
{: .doc_title} 

input validation
{: .doc_subtitle} 

# Summary

Pryv.IO user creation has multiple issues with how it treats the email associated with an account. The validation process is too restrictive, different in Register and Core and sometimes the user input email address is modified before being compared with the stored value.  
We need to treat user input email coherently accross Pryv.IO and not modify it when provided.

**NOTE**: v2 of this design included Email ownership confirmation, which has been extracted to its own design as it is independent from this one.

# Overview

The email address is stored in both Register and Core during user creation and email modification (`account.update`). In both places, the same validation should be done. This issue exists for username validation as well and will be treated in the scope of this task.   

We decide to lowercase the user provided username and email before storing them and before comparing it to the stored one.

Finally, there is an undesirable behaviour in Core for user creation where the API returns an `item-already-exists` error related with the username without checking whether it was the `username` or `email` unicity (or something else) that threw the error.

# Design

The SMTP specification [RFC 5321 - part 4.5.3.1.3](https://tools.ietf.org/html/rfc5321#section-4.5.3.1.3) defines the maximum address name as 256 Bytes. We'll assume that we only US-ASCII characters, therefore we define maximum length to 256 characters.

The following email validation will be modified:  
- Register:[utils.check-and-constraints#email](https://github.com/pryv/service-register/blob/1.2.23/source/utils/check-and-constraints.js#L91)  
- Core: [schema/helpers#email](https://github.com/pryv/service-core/blob/1.3.34/components/api-server/src/schema/helpers.js#L46) 

Regarding the username, we will modify the Register validation to match the Core one which is more restrictive: 25 characters, valid subdomain, which is large enough.  
- Register: [utils.check-and-constraints#uid](https://github.com/pryv/service-register/blob/1.2.23/source/utils/check-and-constraints.js#L58)  
- Core: [schema.user](https://github.com/pryv/service-core/blob/1.3.34/components/api-server/src/schema/user.js#L17)    

The following code in Register modifies (lowercases) user input email addresses, should this modification be done in a way for the intent to be clearer?:  
- [storage.database#emailExists](https://github.com/pryv/service-register/blob/1.2.23/source/storage/database.js#L203)     
- [storage.database#getUIDFromMail](https://github.com/pryv/service-register/blob/1.2.23/source/storage/database.js#L336)  

The user creation code in Core will be modified to return the appropriate error when [inserting a new user in the MongoDB database](https://github.com/pryv/service-core/blob/1.3.34/components/api-server/src/methods/system.js#L46). We will handle 3 cases:    
- username is already taken   
- email is already taken    
- other database errors    

# Test / Quality Control

## Automated

### Register

`POST /users`    
- Modify invalid email in [this test](https://github.com/pryv/service-register/blob/1.2.23/test/acceptance/users.test.js#L102)   
- Add test with some capitalized letters that works  

`POST /users/:username/change-email`   
- Modify invalid email in [this test](https://github.com/pryv/service-register/blob/1.2.23/test/acceptance/users.test.js#L532)   

`POST /email/check`   
- Modify invalid email in [this test](https://github.com/pryv/service-register/blob/1.2.23/test/acceptance/email.test.js#L26)  
- Add test with some capitalized letters that matches a lowercase stored address  

`GET /:email/check_email`   
- Modify invalid email in [this test](https://github.com/pryv/service-register/blob/1.2.23/test/acceptance/email.test.js#L53)  
- Add test with some capitalized letters that matches a lowercase stored address  

`GET /:email/uid` Add tests to [test/acceptance/email.test.js](https://github.com/pryv/service-register/blob/1.2.23/test/acceptance/email.test.js)    
- Succeeds if email is taken  
- Succeeds if email is not taken  
- Succeeds if capitalized email matches an existing lowercased email  
- Fails if email is not matching the required  

### Core

`POST /system/create-user` Add tests for other errors   
- Fails if the email is already taken  
- Fails because database breaks? might be hard to simulate    

# Plan (time estimate)

We intend to modify the following artefacts:

- Register:  
  - Write tests (3h)  
  - Modify email validation (15min)  
  - Modify username validation (15min)  
  - Remove email modification (15min)  
  - Review (1h)
  - Release (30min)

- Core:  
  - Write test (3h)
  - Implement (2h)
  - Review (1h)
  - Release (30min)

The following documentation needs to be updated:

- API reference?: (TBD)	
	- initiate register API docs
- Pryv.IO manual?: (TBD)  
	- initiate  
	- user creation  
		- fields validation  
		- known running conditions (aka bugs)?
		- email: welcome & password reset
		- invitation tokens
	- change email  
	- email usage verification  
	- get username for email  




