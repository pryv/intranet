|         |                |
| ------- | -------------- |
| Author  | Kaspar Schiess |
| Version | 1 (25.9.2017)  |



# Share Access via URL

Create an easy way to share access into a Pryv cloud

* **Who**: A non-technical User
* **Why**: Easy access to a Pryv, without understanding and thinking about all the details that entails. 

Existing Material: 

* Little, we've got this: https://github.com/pryv/lib-javascript/blob/master/source/Connection.js#L359-L371

Discussion: 

* Beginning of Sept, Perki&Kaspar
  * We should just use our sharing url format, it contains the neccessary parts. 
  * We don't see how in the future this is going to bite us, so we decide to go ahead with this for the PowerBI connector. 


* 16May17, (Perki, Ilia, Kaspar):
  * We should have a way of handing auth data around
  * A side goal might be to not forget about details during transfer, like 'http' vs. 'https'
  * There might be two ways to share access; one for handing over access information internally (in code), one for handing over access info from one screen to the next. These would probably be based on a URI schema, both. 
  * If we're to do design on this, we need a catalogue of all the problems this is supposed to solve. Here are the problems we've discussed today: 
    * Handing auth/access info from the iOS application to the Bridge Library. 
    * Handing access information from somewhere into the Tableau connector
    * Allowing to connect the Browser to a slice on another Pryv, to aggregate data. 