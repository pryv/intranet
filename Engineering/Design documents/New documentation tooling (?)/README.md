# Pryvs New Documentation

A list of documentation tools and generators that we know about and that we'll choose from. Samples for the generated directory structures can be found [here](https://github.com/pryv/internal-documentation_samples).

## mkdocs

http://www.mkdocs.org/

Start using 

```bash
cd mkdocs
mkdocs serve
```

Then navigate to http://127.0.0.1:8000/ using your browser. Documents will live-update when changed.

Advantages: 

* Markdown
* Simple, easy to extend 

## API Blueprint

https://www.mulesoft.com/lp/ebook/api/building-api-blueprint

Commercially supported markdown dialect. Limits what can be done quite a bit, and even for simple things one needs to go back to the spec to see how it is done. 

Advantages: 

* Made for APIs 

## RAML

See API Blueprint. Limiting and locked in. 

## Read the Docs

https://readthedocs.com/

Navigate to the website and create an account. This will allow you to demo the functionality. Uses [Sphinx](http://www.sphinx-doc.org/en/master/) under the hood ([github](https://github.com/sphinx-doc/sphinx/commits/master)). A very active project with a commercial front. \$ 150/month

Advantages:

* Hosted service with github build hooks. 
* Keeps old versions around
* Markdown
* Looks very useful, for example http://docs.python-requests.org/en/master/
* Buy

## Sphinx

Plus [recommonmark](https://github.com/rtfd/recommonmark) to support markdown (how to [install](https://stackoverflow.com/questions/2471804/using-sphinx-with-markdown-instead-of-rst#2487862)).

Advantages: 

* Same as Read the Docs minus the monthly price + some customisation that we do on our own. (build)

Killer: Doesn't do markdown tables, because recommonmark -> CommonMark -> 'Standard Markdown'...

## Gitbook

https://gitbook.com

Git backed documentation tool, has an online editor and can be checked out and edited offline. Versioning support. Very much github inspired. About \$ 49 / month.

Advantages

* Git backed
* Clean look
* Markdown

## Pandoc

http://pandoc.org/ 

A general purpose document converter. Might need some setup, but will probably outperform the other tools in terms of output formats. 

Advantages

* Large set of options, configurable
* Does many output / input formats. 
* Depending on our setup, this can be a simple and really good option.

## Readme.io

https://dash.readme.io/

A service that allows creating documentation. Starts at \$ 199 for us. The editor is worse than Word, it makes you click a lot even for simple things. 

Advantages: 

* Has API documentation with code snippets built in. 
* API Editor

## Slate

https://spectrum.chat/slate

Simple tool to generate two-column layouts like Stripe or Paypal has. Markdown. Written in Ruby. Probably not so easy to redesign completely, but then again none of the other tools do that either. 

A sample can be browsed here: http://lord.github.io/slate/?python#

Advantages: 

* Simple, Git based workflow. 
* Could be combined with something like pandoc for larger bodies of text that are non-API documentation.
* If we build something on our own, we should copy the markdown conventions from this tool, somebody has thought some about this.  

## Other Stuff (Not worth a Section above)

*  *dexy* - Old, but really good. Too bad it is not developed/documented anymore. 
* *[devdocs](https://github.com/Thibaut/devdocs#quick-start)* - Probably really good, but demonstration page is blank. 