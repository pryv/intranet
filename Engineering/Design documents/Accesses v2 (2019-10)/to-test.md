# Some use cases to test

## "Read" in  the scope of a "Manage"

### Tree

```
Root
|- A
   |- B
   |- C
   |- D
   		|- E
   			 |- F
```

### Permissions

```yaml
permissions:
	- streamId: A
		level: 'manage'
  - streamId: B
  	level: 'read'
  - streamId: E
  	level: 'none'
```

### Tests

- Can add events to A
- Can add events to B
- Can add streams to A
- Cannot add events to B
- Cannot update B
- Cannot add events to B
- `streams.get` return a Tree with A, B, C, D
- Nothing is allowed Bellow and and including E

### Questions

- Do we rely on the order of permissions ? Or do we re-arrange order by match the Tree?

