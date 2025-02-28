

###  ### Agenda

#### service level

- define role / scope des services

  - reg: deal account creations / directory user - machine 
  - service discovery: 
  - access: obtention d'access (consent) pour un user unkown en début de process. Endoint oAuth
  - core (api): API to manage user data, accounts ….

- minimal set of informations for a service discovery

  - service name
  - access: url / init an oAuth
  - register
  - endpoint API: can take {username} templating 
  - service home page url
  - terms url
  - support url

- optional set

  - logo 
  - user home url: can take {username} templating 

  

##### format sharing 

- usage 
  - Sharing sent (from browser)
  - Campaign Manager: the data processor receives a set of username/domain/token
  - Send connection params to a (stupid) device or service
- sequence Lib or App
  1. decode - API endpoint + Access 
     - api: 
     - auth: 
  2. extract API endpoint from decoded sequence
  3. (optional) retrieve service infos
     1. To get name
     2. terms
     3. support
     4. home 



# Unifying Pryv API & Services entry points

In this document I collect various notes and remarks to engage a process of

- defining methods and references to develop application connecting to Pryv.io API

- modifiying our Docs, Apps, Libs and Demos accordingly

## API

To access pryv resource only 2 information is needed.

- an **API endpoint**  
  Usually on the form of `https://{username}.{domain}:443/`

  But we would like to have more latitude and use the DNS-less capabilities or provide extend pryv data format to publicly available data as exemples.

  - `https://{hostname}/{username}` for a DNS-less set-up
  - `https://api.weather.company/weather/lausanne` for an open data source

  We could also consider the upcoming need of versioning our api and go for

  - `https://{username}.{domain}:443/v2/` 

- an **Authorization** 
  As of today, passed to the API by `?auth=`URL query param or as a header.

#### Authentication process

It's planned to change this process in the future and we should prepare our library, demo apps and customer for an upcoming change. In order to do so we should have a single endpoint of refference to define where to start.

### Actual state and practices are scalable 

We can see a lot of flavors in apps and libs to initiate a Pryv connection. 

Exemple: 

-  **Plotly** : `https://pryv.github.io/app-web-plotly/?username={username}&domain={domain}&auth={token}`
- **Plotly**: `https://pryv.github.io/app-web-plotly/?pryv-reg={register hostname}`
- **App-Web-Access**: `http://pryv.github.io/app-web-access/?pryv-reg={register hostname}`
- ... Each app seems to have is own way 

The **most unrelayable implementation award** could be given to the **lib-javascript** with an overcomplicated init sequence and deprecated on that is still heavily used.

``` json
function Connection() {
  var settings;
  if (!arguments[0] || typeof arguments[0] === 'string') {
    console.warn('new Connection(username, auth, settings) is deprecated.',
      'Please use new Connection(settings)', arguments);
    this.username = arguments[0];
    this.auth = arguments[1];
    settings = arguments[2];
  } else {
    settings = arguments[0];
    this.username = settings.username;
    this.auth = settings.auth;
    if (settings.url) {
      var urlInfo = utility.urls.parseServerURL(settings.url);
      this.username = urlInfo.username;
      settings.hostname = urlInfo.hostname;
      settings.domain = urlInfo.domain;
      settings.port = urlInfo.port;
      settings.extraPath = urlInfo.path === '/' ? '' : urlInfo.path;
      settings.ssl = urlInfo.isSSL();
    }
  }
```

We can even find some implementation directly changing Pryv domain of the lib by `Pryv.defaultDomain = …`which generates further inconsistencies. 

Implementation of cocoa lib being quite similar. 

### Proposal

Define a unfied way to access a Pryv ressource, one simple way would be to rely on initializing a connection with a single URL. 

Exemples:

- `https://toto.pryv.me/?auth=xyz`
- https://toto.pryv.me/#/sharing/xyz` inspired from the sharing link of the browser

Libaries and app connection should be initalized with the same parameter syntax 

Exemples:

- `?ressource=https://toto.pryv.me/%3Fauth=xyz`

## Service

For many of the application the first step is to Authenticate a user, and for this knowing the path to `https://access.{domain}/` is necessary. 

Most implementation even missuses this point and refer to `reg.{domain}` as it is the same machine. 

The worst effect is that application have to **extract** the domain name from a passed parameter to initialize a connection. 

This leads to redundant, not consistent, not documented code in most of the apps that are usually code first with hard coded domain, and fixed after for multi domain capabilities.

Most of our customer will need to disseminate the same information to their Apps.

### Proposal

`https://reg.pryv.me/service/infos` already exposes this concepts

```json
{
"version": "0.1.0",
"register": "https://reg.pryv.me:443",
"access": "https://access.pryv.io/access",
"api": "https://{username}.pryv.io/",
"name": "Pryv Lab",
"home": "https://pryv.com",
"support": "http://pryv.com/helpdesk",
"terms": "http://pryv.com/pryv-lab-terms-of-use/"
}
```

A library or app that needs authetication should be initiated with such an url.

Exemple (or param based)

```javascript
pryv.Auth.setup('https://reg.pryv.me/service/infos', authSettings);
```

And not

```javascript
/**
 * retrieve the registerURL from URL parameters
 */
function getRegisterURL() {
  return pryv.utility.urls.parseClientURL().parseQuery()['reg-pryv'] || pryv.utility.urls.parseClientURL().parseQuery()['pryv-reg'];
}

var customRegisterUrl = getRegisterURL();
if (customRegisterUrl) {
  pryv.Auth.config.registerURL = {host: customRegisterUrl, 'ssl': true};
}


```

#### Other suggested info

- **expires** To make indicate this could expire (cache-until)
- **who-am-i** Valid URL for web-browser SSO. (current set up is not consistent)

### Spreading service infos 

A Pryv ressource connection, initally set-up with a single line url could need to have more info on the service. For example an App might want to "extract" the username from the api or display the Application web site. 

#### Proposal

Relay (and cache) the `/service/infos/` route from all Pryv api endpoint. 

Exemple:

`https://toto.pryv.me/service/infos` => `https://reg.pryv.me/service/infos`	

