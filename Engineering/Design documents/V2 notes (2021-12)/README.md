|           |                      |
| --------- | -------------------- |
| Authors   | Simon Goumaz         |
| Reviewers | Pierre-Mikaël Legris |
| Date      | 2022-01-26           |
| Version   | 1                    |


# ==WIP== Pryv.io v2 design notes


## Motivation

Despite the addition of many features (high-frequency, audit, MFA, etc.), the basic design and architecture basically hasn't changed in almost ten years. With demands from customers taking priority, understandably little has been done to evolve our core implementation in the direction of more decentralization and privacy – a step in the opposite direction was even made by regrouping data across users to improve performance, and it's arguable the influence of complying with norms that were established assuming classic client-server architecture (HIPAA, GDPR) didn't help either.

On the other hand, quite some work we identified as necessary (such as data aggregation/OAuth, fixing issues with stream structure, data encryption) seem to point at the necessity of a wider redesign.

Even if no actual work is undertaken in the short term, we find it important to at least draft a consistent vision for Pryv.io's evolution, taking into account the current state of technology.


## Proposal overview

### Basics that don't change

- Data is siloed per user (owner entity) (this is actually a change back from our temporary implementation at the time of writing)
- Pryv.io nodes keep running on trusted & controlled machines (but there should be a path to, at some point, releasing this constraint, at least on the technical level) ==Q: is this a actually a necessity for compliance with data privacy norms?==
- The entirety of each user's data (her _account_) is hosted by one single Pryv.io node; each node can host multiple accounts
  - This differs from most of today's decentralized networks, which disseminate data across nodes according to privacy, security and performance constraints. As we stick to a trusted & controlled network of nodes ==(see related question above)==, we can simplify things (and also make performance a lot more predictable).
  - This does not prevent the backing up of data by other nodes for (extra) security
  - This also does not prevent clients from maintaining (potentially partial) copies of user data

### Open and generalize the user data model: ontologies→terms + collections→instances

See also [related Miro diagram](https://miro.com/welcomeonboard/anRlVEZVR0dUTWkydzBHYldMejlHQldZTHBMSkxmVTh4TVR2SmRSU2JTc3RTYTFmWUNnbjhablVSU1cxak9KcXwzMDc0NDU3MzY2ODE1NDE4NzIz?share_link_id=237233739581)

==Q: what to name the generalization of streams and their instances? ontologies/metadata/classifications/categorizations/vocabularies/taxonomies/hierarchies… → terms/words/classes/categories/tags/labels/groups…==

Generalize the current _streams_ to any number of _ontologies_ defining _terms_, and generalize the current _events_ to any number of _collections_ containing _instances_.

- Ontologies and collections can be defined in the platform configuration or by apps. The structure (type) of their terms and instances is defined by JSON schemas (standard or proprietary/platform-specific) specified or referred to in their definition.
  - Data structure (type) and its validation is thus done at the ontology and collection level (currently: done at the instance level for events, while streams structure is fixed).
  - Each ontology or collection can accept one or more type(s) of terms/instances
  - It can also then define indexes and constraints (e.g. unicity) for performance and integrity. Indexes and constraints should be defined alongside – not as part of – type definitions (they're outside the scope of JSON schema)
- Collection types can optionally define instances to be linked to ontologies (i.e. we don't force ontologies onto simple use cases that don't need them)
  - In addition to semantics/organization, linking ontologies to collection type enables the application of ontology-based permissions to the collection
  - If no ontology is linked to the type, then only collection (~= type) permissions are available (see more on accesses and permissions below)
- Accesses ==(to be described further elsewhere here)== can define permissions on:
  - Data linked to ontology terms, i.e. the evolution of current streams permissions, with the notable difference that this must be distinct from permission on the terms structure itself (see next point; cf. [request from Addmin](https://github.com/pryv/intranet/issues/26)), although a 'read' permission here automatically implies a 'read' permission on the terms themselves (==right?==)
  - Ontology terms themselves, i.e. to manage terms structure (currently not distinct from the previous point)
  - Collections; this is a new feature even though we've been discussing "type" permissions for a long time (just note that collections are not types, even though there may be a 1:1 correspondance in many cases)
- Include ontology "streams" and collection "events" by default (required for migration)
- Collection types can define instance content to be stored inline (as part of the instance itself, i.e. in the JSON/DB columns) or externally. Externally-stored content is accessed via its own separate API, and includes:
  - Files, accessed via basic HTTP (let's not reinvent the wheel), and possibly located elsewhere entirely (e.g. Amazon S3)
  - High-frequency series, accessed via its REST API (→ InfluxDB)
- ==Q: how to implement first-class relationships between instances?== (identified customer need)
  - Spec: just refer to linked collection(s) in type definitions
  - Big question: what to do when linked items change (are deleted)? Ignore (callers are responsible for maintaining integrity), check & warn, …
  - Example use cases: "this measure was derived from these other ones"; "those pieces of data were collected during that doctor examination"; it must also make it easy to describe graph relationships
- ==Double-check how support for multiple stores (a.k.a. dynamic mapping) integrates with this==

Rationale/comments:
- Storage is better aligned with data structure (the current structure of "everything in the same 'events' bucket" was designed for Pryv's B2C needs, in particular the browser)
- Generalizing streams to ontologies also seems useful in order to provide first-class "semantization" / linking to external ontologies (e.g. SNOMED /LOINC codes), as currently offered by SemPryv
  - To state things like "all `mass/kg` measurements in this stream were obtained by electronic scale" ==Q: to discuss further (is this really desirable / what's the benefit? I [SG] am tempted to think that such data should remain tied to instances/events themselves)==
- Better performance (thanks to ad-hoc collections per data type)
- Better discoverability of data structure, an identified customer need
- Clients must directly query the collections they're interested in (instead of "just" querying _events_ and possibly filtering by type), which is fine in practice as no one should want data they can't understand. And they can always perform multiple queries at once with batch calls.

### ==Ongoing thinking about platform/app data and type definitions==

- Goal: let connecting systems/apps safely assume that some ontologies/collections are present on each platform/app user account and can thus be readily used
- Q: Could we just use API sugar to achieve that? i.e. simply respond with empty data if ontologies/collections don't exist, and create them on the fly when requested to add data to them?
- In any case: the definitions for ontologies/collections must be available somewhere with unique identifiers for platform/apps to refer to
  - Could they be anywhere and the URI be the id? Types could be…
    - either defined locally in platform config and served by a dedicated component (typically for custom types)
    - or served by Pryv or other servers on the internet (typically for standard/common types)


### Replace system streams with 1) first-class account properties and 2) platform data

Account properties are stored with every account and accessed with dedicated API endpoints.
- System account properties (e.g. username) can be extended with custom account properties defined in platform configuration ==(use JSON schema? should be as similar as possible to the rest of the type system – ontologies & collections)==, on par with features currently offered by "account" system streams (more on that below)
- Account properties are read and updated via "account" API endpoints (similarly to the original Pryv API)
- Access permissions can be granted to third-parties via dedicated ACL (defining path(s) to the desired properties and level)

Platform data can be defined in account configuration, similarly to what's currently offered by "other" system streams.

Rationale:
- System streams were meant to provide different things:
  - Enable access control management on user account properties (previously only accessible by "personal" tokens i.e. account owner)
  - Allow defining custom account properties with optional enforcement of rules at the platform level, such as: unique, indexed (i.e. available for querying via the system API), required for creation, editable after creation ("account" system streams in [doc](https://api.pryv.com/customer-resources/system-streams/#about-system-streams))
  - Allow defining platform-level streams (i.e. the same for all users; "other" system streams in [doc](https://api.pryv.com/customer-resources/system-streams/#about-system-streams))
- But usage has shown issues, in particular that streams and events are not appropriate constructs to model user account properties – the latter are by nature separate from user streams and events and must be dealt with as such

### Make apps first-class constructs

==To expand== App registration, management, app data – see also OAuth 2.0 design doc and description of platform/app above

### Fully decentralize all functions

There is only "core" nodes, with platform-level data replicated P2P among them.

- Register/DNS (which may no longer be DNS?)
    - DNS is a simple "pre-flight" . It has been proven to be very efficient, but also a "blocker" in some cases. 
    - We should investigate if they are some standard than are covering this feature.
- P2P DB implementation: see [etcd](https://etcd.io/)

Rationale:
- Much simpler platform management (just one role for every machine)
- 

### Implement end-to-end data encryption at the core

==More research needed; see also [Perki's experiment with proxy reencryption](https://github.com/perki/test-proxy-re-encrypt)==
- Need a flexible/pluggable architecture allowing different kinds of encryption

### ==Points to investigate further==

Tied to the changes on data model and data encryption (and also indirectly full decentralization):
- How to support platform-level "discovery" queries (via indexes) such as: ==⇒ "app", could run on core so that data never leaves the machine==
  - "count all people over 40 that have diabetes" and "send them a consent request for data X"
  - "list all users who consented to sharing stream/type X"
  - "list users per social welfare number"
  - …and make this work consistently with privacy / end-to-end data encryption

### Other changes ==to be expanded==

- Each piece of addressable data (account, stream, event) is identified by a URI (p.ex. URL `pryv://{user-id}/streams/{stream-id}`) – we're almost there, just missing the `pryv` scheme
- Implement transactions (sequence of actions/low-level events that are then committed and broadcast at once) ↔︎ batch calls (option?)
- Notifications include the full data (in batches when possible, cf. transactions) ==to make this simple requires data to be encrypted==; dedicated notification components can forward them to federated services (Apple, Google etc.) and/or webhooks
  - On the more general topic of messaging: both Axon and TChannel (used in service core as of 2022-06-20) are no longer maintained; we should replace both e.g. with [gRPC for Node.js](https://github.com/grpc/grpc-node/) as part of a redesign there
- Implement "global" constraints and functionalities for a platform. (settings i.e. model, actions) for all user accounts of a platform ==_SG: Perki, I don't see what you mean here that is not covered by the ontologies/collections definitions, account properties and platform data features described above?_==
- Events query: support content search, …

### Dismissed ideas

- ~~Address data by content instead of random ids? Maybe not of much use as long as we store each account in one single place (a big draw of content ids is they allow deduplication of data), and costly in performance~~


## Inspiration

From the current state of technology; trying to focus on projects with related aims.

### Textile's ThreadsDB ([docs](https://docs.textile.io/threads/), [white paper](https://docsend.com/view/gu3ywqi))

Threads is a "protocol and decentralized database that runs on IPFS meant to help decouple apps from user-data", and is thus based (like Pryv.io) on the premise of user-siloed data.
- Rough model:
  - A _Thread_ is basically a decentralized container of _Databases_ for a given user
  - A _Database_ stores _Collections_
  - Each _Collection_ defines one or more schemas, and can only store _Instances_ (i.e. records, documents etc.) that match those
- 

### [Ceramic](https://ceramic.network/) ([dev site](https://developers.ceramic.network/))

IPFS-based "stream processing network" that provides "mutability, version control, access control and programmable logic" to build "fully-featured decentralized applications."
- Streams are the primary construct, each stored as a DAG with an immutable id and verifiable state (a.k.a. current value), similar to a git tree with each change being a new commit.
- Each stream defines its type, i.e. a function enforcing rules and logic to new commits (data structure, content format, authentication/access control, consensus algorithm…)
- Authentication mechanisms are open but the primary one provided is decentralized identifier ([DID W3C spec](https://www.w3.org/TR/did-core/)).
- Each Ceramic network is made of a set of nodes running the protocol and communicating via [Libp2p](https://libp2p.io/). Each node persists the state of streams it is "pinning" (multiple nodes can – and should, for resilience – maintain each single stream). It forwards queries about streams it isn't pinning to the rest of the network (returning the response itself).
- Security is provided by signing commits by the stream's "controller" and/or by publishing content on a blockchain ("proof-of-publication", using Ethereum it seems).

Comments:
- Ceramic looks more like a base-layer framework than a ready-to-use middleware. Most hard decisions are externalized to the stream types, for example:
  - Data structure. The existing types offer typing based on JSON schema, but apart from that apps have to do it all (hierarchy, classification…)
  - Permissions management. Same.
  - Consensus. The implementation for existing types is functional but offers no protection against malicious nodes (basically json-patch diffs with first-update-wins).
- Also, almost nothing is offered about data resilience (no keeping a minimum number of copies of data across the network, etc.). Nodes join, leave, pin streams, cache etc. all manually.

### Not considered

- Anything primarily based on blockchain (e.g. Filecoin…)
- (ex-Dat) Hypercore (append-only logs), Hyperdrive (files), Hyperbees (key-value), Hyperswarm (finding peers via DHT); doesn't offer ready constructs for database collections
- Fission (IPFS-based); dev-friendly toolkit with focus on app-siloed data
- Fleek (IPFS-based); dev-friendly toolkit based on other components like Textile & Filecoin
- OrbitDB (IPFS-based); focused on app-siloed (not user-siloed) data
- Safe Network: 


