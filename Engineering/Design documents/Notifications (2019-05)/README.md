|         |                       |
| ------- | --------------------- |
| Author  | Ilia Kebets (Pryv SA) |
| Reviewers | |
| Date    | May 16th 2019            |
| Version | 1                     |

Pryv.io Notifications
{: .doc_title} 

Overview document
{: .doc_subtitle} 

# Summary

This document describes Pryv.io `api-server`'s Notifications flow

## Notifications sending

Upon server setup, the [notifications bus](https://github.com/pryv/service-core/blob/1.4.8/components/api-server/src/server.js#L218) is created.
This one has 2 ways of sending updates:
- in-process: through its [EventEmitter interface](https://nodejs.org/docs/latest-v8.x/api/events.html#events_class_eventemitter)
- to other processes: through the [Axon message queue](https://github.com/pryv/service-core/blob/1.4.8/components/utils/src/messaging.js#L11).

The sending is done in [Notifications.dispatch()](https://github.com/pryv/service-core/blob/1.4.8/components/api-server/src/Notifications.js#L46)

## Reception

### Socket.IO

The Socket.IO server is setup with the [notifications in parameters](https://github.com/pryv/service-core/blob/1.4.8/components/api-server/src/socket-io/index.js#L44) because it receives them in-process.
The socket.IO resends these through [its NATSpublisher](https://github.com/pryv/service-core/blob/1.4.8/components/api-server/src/socket-io/index.js#L60). Finally, the socketIO has a Manager object storing a [map](https://github.com/pryv/service-core/blob/1.4.8/components/api-server/src/socket-io/Manager.js#L54) of `/username` (namespace)/[NamespaceContext](https://github.com/pryv/service-core/blob/1.4.8/components/api-server/src/socket-io/Manager.js#L171) pairs. Each NamespaceContext is created and [subscribes to NATS](https://github.com/pryv/service-core/blob/1.4.8/components/api-server/src/socket-io/Manager.js#L243) notifications upon successful [socket.IO authorization phase](https://github.com/pryv/service-core/blob/1.4.8/components/api-server/src/socket-io/index.js#L91).  



This rather complicated structure ensures that when any `api-server` process receives a change for a user, the socket-IO object broadcasts the change through the NATS message queue.  
As all socket.IO connections for a user are bound to a certain `api-server` process, it is sure to receive it and dispatch it to its client(s).

