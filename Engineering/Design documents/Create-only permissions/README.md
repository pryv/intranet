

# Create-only permissions

In order to allow our customers to use create-only permissions before the accesses v2, we introduce the `create-only` level.

It is allowed exclusively for **stream permissions** - so forbidden for tag permissions.

## Tag-based permissions

The behaviour of tag-based permissions is incompletely tested.

- stream-based permissions should not be allowed to create tag-only accesses
- stream-based permissions should only be able to add tags if keeping a subset of the streams
- tag-based permissions should not be able to create accesses that have additional tags - no shit sherlock
- tag-based permissions should not be able to create stream-only based permission accesses.
- mixed permissions should not be able to create accesses with permissions that are:
  - tag-only
  - Stream-based with additional tags, only a subset of tags is allowed
  - Obviously, if stream-based only, must be a subset of its streams

## Mixed permissions

Should we allow mixing up create-only permissions with other ones? This is a relevant question as it introduces multiple use cases where the behaviour is not obvious

We explore the following setup:

#### Permissions: 

- stream+create-only
- stream+manage



## CREATE

### on create-only permission stream (or its children)

#### Forbidden:

- stream+read

- stream+contribute
- stream+manage
- tag+any -- allowing to create tag-only permissions is a leak! Enforcing to have at least the same stream sounds complicated. 

#### Allowed

- stream+create-only ??

