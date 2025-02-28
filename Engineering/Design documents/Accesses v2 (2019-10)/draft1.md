



Permissions:

Tree 




```
permissionSets:

manage:
      - context.create: true
      - context.update: true
      - context.get: true
      - context.delete: true
      - event.create: true 
      - event.update: true
      - event.delete: true
      - event.get: true

read:
		  - context.create: false
      - context.update: false
      - context.get: true
      - context.delete: false
      - event.create: false 
      - event.update: false
      - event.delete: false
      - event.get: true
      
contribute:
		  - context.create: false
      - context.update: false
      - context.get: true
      - context.delete: false
      - event.create: true 
      - event.update: true
      - event.delete: true
      - event.get: true
  
createonly:
		  - context.create: false
      - context.update: false
      - context.get: true
      - context.delete: false
      - event.create: true 
      - event.update: false
      - event.delete: false
      - event.get: false
      
onlynotes:
		  - context.create: false
      - context.update: false
      - context.get: true
      - context.delete: false
      - event.create: true, type=note/txt 
      - event.update: true
      - event.delete: true
      - event.get: true
      
none: 
		- context.create: false
    - context.update: false
    - context.get: false
    - context.delete: false
    - event.create: false 
    - event.update: false
    - event.delete: false
    - event.get: false



  type: personal
  delegates:
    - app
    - shared
  methods:
		- auth.login: false
    - stream.create:  root
    - stream.get: root
    - stream.delete: root
    - stream.update: root
    - event.get: {streamIds: [root]}
    - event.create: {streamIds: [root]}
    ...
  allowedCustomization: []
,
	
  type: app
  methods:
     - accesses.create: true
     - auth.login: false
     - access-info.get: true
  allowedCustomization:
    streamId: {manage, createonly, none, contribute}, 
    tag: {manage, createonly, none, contribute} ] 

  delegates: 
    shared
  ]
,
  type: shared // idem app
  methods:
  	- accesses.create: false

,  
  
  type: personal_only
  name: Personal Only
  methods:
    auth.login: true
    access.create: true
    streams.get: true
  delegates:
    - app
    - shared
  permissionSets:
  	- auth.login: true
    - stream.create: false
    - stream.get: root
    - stream.delete: false
    - stream.update: false
    - event.get: false
    - event.create: false
  



Exemple of Access.create

============

type: app
permissions: 
  - tag: super
  	level: contribute
  - tag: fragislistic
  	level: read
  	
  	
permissions:
	- streamId: *
		events.get: {allowedTags: [fragislitic, super]}
		events.create: {allowedTags: [super]}
		
		
============

type: app
permissions: 
  - streamId: diary
  	level: read
  	params: type=note/txt
  - streamId: apple
  	level: none
  - streamId: ifttt
  	level: manage
  	
  	
permission
	- stream.create: {streamIdHasAncestor: [ifttt], streamIdDoesNotHaveAncestor, [apple]}
	- stream.get: {streamIdHasAncestor[ifttt, diary]}
	- stream.delete: {streamIds: [ifttt]}
	- stream.update: {streamIds: [ifttt]}
	- event.get: {streamIds: [ifttt, diary]}
	- event.create: {streamIds: [ifttt]}
	
	============

type: app
permissions: 
  - streamId: diary
  	level: read
  - streamId: apple
  	level: none
  - streamId: ifttt
  	level: manage
  - tag: super
  	level: read
  	
  	
permission
	- stream.create: {streamIdHasAncestor: [ifttt], streamIdDoesNotHaveAncestor:[apple]}
	- stream.get: {streamIdHasAncestor[ifttt, diary]}
	- stream.delete: {streamIds: [ifttt]}
	- stream.update: {streamIds: [ifttt]}
	- event.get: {streamIds: [ifttt, diary], tags: ['super']}
	- event.create: {streamIds: [ifttt], tagIN: ['super']}
  	
  	
 


```

