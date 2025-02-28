
# Pryv.io data integrity

## Motivation

Provide integrity checks to attachment files.

### cards

- [Provide data integrity to Pryv.io](https://trello.com/c/DUMx0FSz)

## Design

### API

This feature would modify the following API calls:

- [events.create](https://api.pryv.com/reference/#create-event) (with an attachment)
- [events.addAttachments](https://api.pryv.com/reference/#add-attachment-s-)

It will bring a read-only checksum property to the [attachment](https://api.pryv.com/reference/#event) data structure.

Once uploaded, the client can verify that the server has stored the file correctly by comparing the hash.

## Discussion

- we're not providing it in the request as the underlying TLS protocol provides integrity.

- what hashing algo? MD5, sha-256, CRC32?
  - requirements: integrity, speed, ease of use client-side

- what field to attachment? `checksum`? `integrity:sha-256-checksum`?
  - requirements: it should contain the hashing algorithm so we can easily change it.
  - we're dropping `digest` because it is confusing with `HTTP digest authentication`
  - `integrity: "sha-256-${Base64(hash)}"`

## Scope

- Migrate attachments into user storage

## References

- [subresource integrity](https://www.troyhunt.com/protecting-your-embedded-content-with-subresource-integrity-sri/)
