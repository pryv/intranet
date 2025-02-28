|         |                       |
| ------- | --------------------- |
| Author  | Ilia Kebets (Pryv SA) |
| Reviewers | Pierre-Mikael Legris (Pryv SA) |
| Date    | May 29th 2019            |
| Version | 2                     |

Metaform
{: .doc_title} 

Pryv.io demo integration
{: .doc_subtitle} 

## Summary

This document describes the way we propose to integrate the Metaform app with a Pryv.io backend for the scope of a demo.  

This version is a proposal based on discussions up to now and is subject to change.

# Data structure

This part describes Metaform's data structure and its mapping into Pryv.io's data structure: [Streams](https://api.pryv.com/reference/#stream) and [Events](https://api.pryv.com/reference/#event).  

## Representational data structure

~~~~
Metaforms
  |_Projects
    |_Project
      |_Form
        |_(Record)
          |_Field
          |_Field
          |_Entity
            |_Field
            |_Field
        |_(Record)
          |_Field
          |_Field
          |_Entity
            |_Field
            |_Field
      |_Form
        |_(Record)
          |_Field
        |_(Record)
          |_Field
~~~~

Metaforms objects will have a 1-to-1 correspondance with Pryv.io Streams, except for Records which will be represented by Tags on Pryv.io Events.

At the root of the Pryv.io streams, you have a stream `metaforms`, which has a single child `projects`.  
The `projects` stream has multiple child streams `project-id` where id is a unique project id from Metaforms.  
Each `project-id` stream has multiple child streams `form-id` where id is a unique form id from Metaforms.  
Each `form-id` stream has multiple child streams `field-id` or `entity-id` where id is a unique field or entity id from Metaforms.  
Each `entity-id` stream has multiple child streams `field-id` where id is a unique field id from Metaforms.  

Finally, `field-id` streams contain events that store the field values and `record-id` as following:  

~~~~
streamId: `field-id`
type: to be defined
tags: [`record-id`]
content: fieldValue
~~~~

## Mapping per data structure

The Metaforms app will write data under the root stream `metaforms`.

The following tables and examples describe the mapping of Metaform objects into the Pryv.io data structure:  

### Project

All project streams will have the `parentId` field set to `projects`.  

|Metaform property|Pryv property|
|-----|---------|
|name|name|
|id|`project-`id|
||parentId=`projects`|
||clientData.metaform:type=`project`|

#### Example

Pryv.io Stream:  

~~~~~yaml
name: Patient body parameters
parentId: projects
id: project-${mf-project-id}
clientData:
  metaform:type: project
~~~~~

### Form

|Metaform property|Pryv property|
|-----|---------|
|name|name|
|id|`form-`id|
|project.id|parentId|
||clientData.metaform:type=`form`|

#### Example

Pryv.io Stream:   

~~~~~yaml
name: Patient weight
id: form-${mf-form-id}
parentId: project-${mf-project-id}
clientData:
  metaform:type: form
~~~~~

### Field or entity

|Metaform property|Pryv property|Note|
|-----|---------|--|
|name|name||
|id|`field`-id||
|form.id|parentId||
|entity.id|parentId|in case it is in an entity|
||clientData.metaform:type=`field`/`entity`|

#### Example

Pryv.io Stream:   

~~~~~yaml
name: Weight
id: field-${mf-field-id} or entity-${mf-entity-id}
parentId: form-${mf-form-id}
clientData:
  metaform:type: field or entity
~~~~~

### Measurement

|Metaform property|Pryv property|Note|
|-----|---------|--|
|field.id|streamId||
|record.id|tags[0]||
||type=`CLASS/FORMAT`|corresponding to Metaforms type|
|type|clientData.metaform:fieldType|
|value|content||
||clientData.metaform:type=`fieldValue`||

#### Example

Pryv.io Event:   

~~~~~yaml
id: random-string
streamId: field-${mf-field-id}
tags: [ record-${mf-record-id}]
type: mass/kg
clientData:
  metaform:type: fieldValue
  metaform:fieldType: fieldType
~~~~~

# Integration flow

In Metaforms:

Sign in as *doctor*, create dataset, form, project. Then assign it to a patient.  
When a patient signs in to Metaforms, the web app fetches his Pryv.io credentials username/token.  
Once a user fills a form and clicks on `save`, the required Stream tree and events are sent to Pryv.io.

You will find some example NodeJS code on how to perform the save [here](https://github.com/perki/test-metaform).
