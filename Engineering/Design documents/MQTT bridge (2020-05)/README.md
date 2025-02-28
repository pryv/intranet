|         |                       |
| ------- | --------------------- |
| Authors | Ilia Kebets |
| Reviewers | Simon Goumaz (v1), Pierre-Mikael Legris (v1) |
| Date    | May 11th 2020 |
| Version | 2                  |

# MQTT bridge for Pryv.io

## Motivation

As multiple customers have sensors using the MQTT protocol, it is useful that Pryv.io offers a compatible interface.

## Proposition

We propose to implement a bridge that will stand in front of the Pryv.io API and proxy MQTT calls.

We will map data coming from the bridge exclusively to HF points.

## Deliverables

### API

- Support for MQTT protocol

### Docs

- mqtt.login
- mqtt.createPoints
- mqtt.logout

### Infrastructure

- Bridge running the bridge server

### Client

- connect
- send data points
- logout

## Design

The MQTT bridge public interface will act as a **MQTT broker**, accepting connections and data points publications from clients.

Another service will be verifying credentials against the Pryv.io API and proxying data points for the broker.

### Client connection

1. In order to start sending data to Pryv.io, a client will provide its credentials upon sending the **Connect** message providing a *pryvApiEndpoint* in the `username` field. See [App guidelines](https://api.pryv.com/guides/app-guidelines/) for pryvApiEndpoint format
2. The bridge service will receive credentials, verify them against the Pryv.io API using a **getAccessInfo** call.
  2.1 If the credentials are valid, it will return a code 0 in the **Connack** message, allowing the client to publish topics prefixed with the client's id
  2.2 If the credentials are invalid, it will return a code 5 (unauthorized)
3. Authorized clients will be stored in a **clientId-pryvApiEndpoint** pair.

### Receiving data points

Once a client is authorized, it will be able to transmit HF series points based on the payload which must map the one of the [Add HF series batch](https://api.pryv.com/reference/#add-hf-series-batch) method. The bridge service will resolve the **clientId-pryvApiEndpoint** based on the clientId.  
We are currently not using MQTT topics for anything.

As we will not queue messages to subscribers, we will not use a persistence for data points.

#### QoS

We propose to ensure all 0,1,2 QoS levels. As the listener will be the Pryv.io HTTP API, it will receive the message every time. The level will depend whether the client wants to be notified of the sending of the message.

Ref.: [MQTT QoS](https://www.hivemq.com/blog/mqtt-essentials-part-6-mqtt-quality-of-service-levels/)

### How do we add TLS, rate limiting and DoS mitigation?

using [NGINX](https://www.nginx.com/blog/nginx-plus-iot-security-encrypt-authenticate-mqtt/).

## Implementation

We propose to use [mqtt](https://www.npmjs.com/package/mqtt) for the client and [mosca](https://www.npmjs.com/package/mosca) ([wiki link](https://github.com/moscajs/mosca/wiki)). For the bridge-service, we wish to use the [Pryv JS lib](https://www.npmjs.com/package/pryv) to address the Pryv.io API.

## Test Cases

- mqtt.connect
  - when using valid credentials
    - should return a code 0
    - check database for credentials
  - when using invalid credentials
    - should return a code 5
- mqtt.createPoints
  - when authorized
    - check return code 0
    - should save these points on the related Pryv.io account https://api.pryv.com/reference/#get-hf-series-data-points
  - when unauthorized
    - should return an error
- mqtt.disconnect
  - when authorized
    - should return some way to accept it
  - when unauthorized
    - should return some kind of error

## Discussion

1. How do we verify that the HF events the client publishes to are in his access' permissions list? we currently don't
2. We might omit the usage of the connect method to allow our broker to proxy publish messages from unauthorized clients to the Pryv.io API.

### How do we transmit the Pryv.io API errors to the client?

The current implementation is based on using the [`published`](https://github.com/moscajs/mosca/wiki/Mosca-basic-usage#receiving-data-from-clients) event to get the message's content and send it to the Pryv.io API.  
I'm not sure how it is possible to provide some response to the client's message publication.

## Out of scope

TBD
