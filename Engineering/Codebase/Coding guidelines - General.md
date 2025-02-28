# General coding guidelines

Those guidelines apply to all applicable situations, except 1) if mentioned otherwise in specific guidelines or 2) if widely-used standards exist and differ.

Specific guidelines documents should follow the same sections order.


## Testing

All implementation code that can reasonably be covered by automated tests should be covered by automated tests.

- Functional (or acceptance) tests are required; they are the only functional specifications we keep of completed work
- Unit tests are only required if they provide clear value (e.g. as documentation) in addition to the functional tests
- Always work test-first if applicable; if not make sure you can explain why


## Files and folders

### General

- Naming:
	- Code files should be named after the module/class they expose (e.g. file `GrooveGenerator.js` exposes class/constructor `GrooveGenerator`, file `errorHandling.js` exposes helper functions namespace `errorHandling`)
	- Other files and folders should generally follow `lower-case-with-dashes.extension`
	- Avoid redundancy in names: if the context of a file (e.g. its containing folder) is clear, don't make it part of its name (e.g. use `storage/Events.js` instead of `storage/EventsStorage.js`)
- One module/class per file, with matching names

### Usual organization

- Source (i.e. implementation) code goes into folder `source` (or `src`), or for specific cases: `lib` (library), `server`, `app`, etc.
- Test code goes into `test`
	- Unit tests structure should mirror implementation files structure: for example, test file for `{source}/folder/module.js` is found in `/test/folder/module.test.js`
	- Integration tests are found in clearly named folders like `/test/acceptance`, `/test/load`, etc.
- Static assets (such as media content or CSS) go into `{source}/assets`


## Code

### Naming

- Functions should be named like verbs, e.g. `doSomething()`
- Booleans variables and properties should be named like adjectives, e.g. `active`
- Arrays, list, collections etc. should be named using the plural form (e.g. `potatoes` is a list of potato species)
- Use descriptive names that are self-explanatory in their context
	- Be explicit and precise: the priority is readable code, not short code (where short code matters there are usually minifying tools for that)—for example, you should generally prefer `responseStream` to `rs`, or `propertyName` to `prop`. Exceptions to this include common cases in contexts where shortened names are used by convention (e.g. `err` for error, `res` for HTTP response, etc.)
	- ...but as for filenames, don't be redundant: if the context of an item is clear, don't make it part of its name
	- Keep very condensed names like `e` or `desc` for short local scopes
- String identifiers (e.g. for errors) should be `lower-case-with-dashes`
- Do not capitalize the 'Y' in our name; just use `pryv` or `Pryv` (i.e. not `PrYv`)

### Code layout / style

- Maximum line length: 100 characters
- Indentation:
	- Use spaces (soft-tabs), not tabs
	- 1 level = 2 spaces
- Order declarations for best readability: higher-level code first, then implementation details, in usage and reading order ("good code should read like a book")
- Spacing:
	- One instruction per line
	- Use blank lines to separate groups of related instructions ("paragraphs" of code)
	- No extra spaces at the end of lines (including blank lines)—this should be enforced by your editor or a pre-commit hook if needed

### Comments

- Only use comments to describe non-obvious code behavior
- When applicable, comments should follow a standard format so that they can be processed by IDEs (to provide coding hints) and doc generators

### Miscellaneous

- Keep functions short and operating within a single level of abstraction
- Use duck-typing and rely on tests rather than testing for types of arguments
- Optimize for expressiveness and readability; only think of performance when there's a measurable problem
- Use core language constructs over library-specific constructs wherever possible


## Source control (i.e. Git) and versioning

- One change per commit
- Clear and explanatory commit messages, always mentioning the related feature or issue  if applicable (use GitHub commit messages integration, e.g. `fixes #321`)
- Tag each published release with its version; usual format is `v{major}.{minor}.{revision}` (example: `git tag v0.4.15`)
- Use [semantic versioning](http://semver.org/)

### Repository naming

General scheme:
- `service-*`: Pryv core service
- `app-*`: Pryv client app, end-user facing
- `lib-*`: Pryv client code library
- `example-*`: Example code
- `dev-*`: Development util
- `ops-*`: Operational util
- `poc-*`: Proof of concept
- `{package name}`: Published package (e.g. on npm); use `pryv-*` for `@pryv/*` namespaced packages


## Documentation

### README

Every repo's `README.md` file should describe (if applicable):

- The repo's nature and purpose (usually very briefly)—this should match the GitHub repo description
- Usage instructions if applicable (e.g. for public code libraries)
- How to setup the development environment
- How to run the tests and debug
- How to package and deploy
- The repo's file organization (if not obvious)
- The license if applicable ([our "documents" repo](https://github.com/pryv/documents) contains the typical license we use for code)

Also include under the top heading, if applicable: [Travis CI build status badge](http://about.travis-ci.org/docs/user/status-images/), published version in package manager (e.g. [Version Badge](http://badge.fury.io)), dependencies status (e.g. [Gemnasium](https://gemnasium.com)).

### CHANGELOG

- Maintain a `CHANGELOG.md` file if other code depends on the project
- The change log should at least contain a section for each published version including new features and/or breaking changes
