|         |                      |
| ------- | -------------------- |
| Author  | Pierre-Mikael Legris |
| Reviewers | Ilia Kebets |
| Date    | 26.03.2019           |
| Version | 2                    |

# Issue

Out of performance tests on user creation made in January 2018.

The following has been measured:

- High Memory consumption of MongoDB with a linear correlation with Number of users (1Mb / user)
- Evaluation shows that this is due to per-index / per-collection memory footprint s

This is not acceptable for customers with > 10000 users with low access rate. As it leads to a non-motivated horizontal scaling.  

Directly impacted customers: Riva, IMAD,

At medium term: 90% of our customers

## Project overview

This project aims to evaluate if we can lower this consumption by redesigning the database. 

Possible solution: remove per-user collections by a single per-object collection. 
i.e.: instead of having an events collection per user to have one single collection "Events" for all users
idem for:  
- Profile
- Streams
- followedSlices
- Accesses

### Note 

This new design implies a change at the technical level that can have a business impact. 
The collections will not be separated anymore on the file system. This makes impossible to delete per-user data on backups, unless restoring and running the MongoDB database. 

### Code pointers

The goal is to get rid of {userid}.{object} collections and have one single collection per data structure.

We distinguish 3 different tasks

1. **#addUserId** for all document adds a `userId` property to separate documents per user:

   1. add `userId` to indexes
   2. add `userId` to all incoming data
   3. add `userId` to all queries
   4. remove `userId` from outgoing response

2. **#useStreamIds** to objects that have non randomly generated (currently cuid-format) `id` fields to Streams and Profiles:

   1. let mongo generate `_id` property and rename internally `id` to `streamId`
      - for incoming data
      - for queries
   2. rename them back for outgoing responses

3. **#deleteToNull** for Streams indexes (as was done for [accesses](../Access%20deletions%20retrieval%20(2018-11-30)/README.md)): 

```javascript
index: { name: 1, parentId: 1 },
options: { unique: true, sparse: true }
```

was the previous index structure

By introducing the `userId` field in indexes

```javascript
index: { userId: 1, name: 1, parentId: 1 },
options: { unique: true, partialFilterExpression: {
     deleted: { $type: 'null'}
}
```

The [sparse option](https://docs.mongodb.com/v3.4/core/index-sparse/) was not usable anymore because deletion simply unset `name` and `parentId` properties, which worked with sparse indexes. Now that we have an always-set `userId` property, we can not rely on sparse indexes anymore. Therefore the uniqueness constraint filtering is handled by the `deleted` property.

As it is not possible to use the `{ $exists: false}` filter, I force all items to have a deleted property (set to null) if none. 

This **Hack** does not satisfy me (Perki)!! 

## Implementation

### Streams, Events, ..

For all objects that were split in different collections. Single Collection Mode is activated by having `getCollectionInfo(user)` returning an object with the additionnal `useUserId = {userId}` property

```javascript
Streams.prototype.getCollectionInfo = function (user) {
  return {
    name: 'streams',
    indexes: indexes,
    useUserId: user.id
  };
}; 
```

### Database

- for all collections' indexes add a `userId` property if the `collectionInfo.useUserId` property exists

```javascript
if (collectionInfo.useUserId) {
    const newIndexes = [{index: { userId : 1}, options: {}}];
    for (var i = 0; i < collectionInfo.indexes.length; i++) {
      const tempIndex = {userId: 1};
      for (var property in collectionInfo.indexes[i].index) {
        if (collectionInfo.indexes[i].index.hasOwnProperty(property)) {
          tempIndex[property] = collectionInfo.indexes[i].index[property];
        }
      }
      newIndexes.push({index: tempIndex, options: collectionInfo.indexes[i].options});
    }
    collectionInfo.indexes = newIndexes;
  }
```

Order is important in compound indexes ([reference](https://docs.mongodb.com/v3.4/core/index-compound/#create-a-compound-index)): 

*The order of the fields listed in a compound index is important. The index will contain references to documents sorted first by the values of the `item` field and, within each value of the `item` field, sorted by values of the stock field.*

Even if order is not garanteed in ECMAscript specifications, we add `userId` at 1st position, because it seems that NodeJS and MongoDB's scripting enforces it. 
This was verified in benchmarking results.

  

- For all methods with a query, the `addUserIdIfneed()` method is called:

```javascript
/**
 * Add User Id to Object or To all Items of an Array
 *
 * @param collectionInfo
 * @param {Object|Array} mixed
 */
addUserIdIfneed(collectionInfo: CollectionInfo, mixed) {

  if (collectionInfo.useUserId) {
    if (mixed.constructor === Array) {
      const length = mixed.length;
      for (var i = 0; i < length; i++) {
        addUserIdProperty(mixed[i]);
      }
    } else {
      addUserIdProperty(mixed);
    }
  }

  function addUserIdProperty(object) {
    object.userId = collectionInfo.useUserId;
  }
}
```

**Note:** this has not be implemented in converters, as they do not have access to the `userId`

  

- Specfic case of `countAll` does not use the [collection.countDocuments](http://mongodb.github.io/node-mongodb-native/3.1/api/Collection.html#countDocuments) function anymore: 

```javascript
this.getCollectionSafe(collectionInfo, callback, collection => {
      collection.find(query).count(callback);
    });
```

- Specifc case of `totalSize()` now also returns document count, as there is no way to have storage size per user: 

```javascript
if (collectionInfo.useUserId) {
      return this.countAll(collectionInfo, callback);
    }
```

- Specific case `dropCollection()` now deletes all documents related to this `userId`

```javascript
 if (collectionInfo.useUserId) {
      return this.deleteMany(collectionInfo, {}, callback);
    }
```


### Base Storage

- `findDeletion()` now passes uses `query.deleted = { $ne: null }` instead of `query.deleted = { $exists: true }`.

- `applyConvertersFromDB()` removes property `userId`

```javascript
function applyConvertersFromDB(object, converterFns) {
  if (object) {
    if (object.constructor == Array) {
      for (var i = 0; i < object.length; i++) {
        if (object[i].userId) {
          delete object[i].userId;
        }
      }
    } else {
      if (object.userId) {
        delete object.userId;
      }
    }
  };
  return applyConverters(idFromDB(object), converterFns);
}
```

Note: this could be moved to some converters

### Events (BaseStorage)

 - In **findStreamed()** in `ApplyEventsFromDbStream`, delete the `userId` property and `deleted` if it is `null`: 

```javascript
delete event.userId;
   if (event.deleted == null) {
     delete event.deleted;
   }
```

### Streams (BaseStorage)

- Use of partial index instead of spare: 

```javascript
index: { streamId: 1, options: {unique: true} },   
index: { name: 1, parentId: 1 },
    options: { unique: true, partialFilterExpression: {
      deleted: { $type: 'null'}
    } }
```

### Converters

Here is an example for [Streams](https://github.com/pryv/service-core/blob/1.4.4/components/storage/src/user/Streams.js#L38), but the same has been done for [Profiles](https://github.com/pryv/service-core/blob/1.4.4/components/storage/src/user/Profile.js#L18).

```javascript
_.extend(this.converters, {
 ...
  itemToDB: [
    converters.deletionToDB,
    converters.stateToDB
  ],
 ...
  convertIdToItemId: `streamId`
});
```

```javascript
BaseStorage.prototype.addIdConvertion = function() {
  if (this.idConvertionSetupDone) return;

  if (this.converters.convertIdToItemId) {
    const idToItemIdToDB = converters.getRenamePropertyFn('id', this.converters.convertIdToItemId);
    const itemIdToIdFromDB = converters.getRenamePropertyFn(this.converters.convertIdToItemId, 'id');
    this.converters.itemToDB.unshift(idToItemIdToDB);
    this.converters.queryToDB.unshift(idToItemIdToDB);
    this.converters.itemFromDB.unshift(itemIdToIdFromDB);
  }
  this.idConvertionSetupDone = true;
};
```


## Tests and performance

This implementation should not impact overall performance of Pryv.io API

Tests to be done: 

- Memory consumption
- User creation rate
- Event creation rate
- Event retrieval rate


### Constraints

- The changes should preserve the capacity to develop per-user storage in the future. As this need has been evaluated.

