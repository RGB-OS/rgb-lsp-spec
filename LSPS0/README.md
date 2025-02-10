# LSPS0 Transport Layer Specification

# LSPS0 Transport Layer

| Name    | `transport`        |
| ------- | ------------------ |
| Version | 1                  |
| Status  | Stable             |

## Motivation
The transport layer describes how a client is able to request services from the LSP.

## Actors
- **LSP**: The API provider, acting as the server.
- **Client**: The API consumer.

## Common Schemas
As the transport layer protocol described in the following uses JSON, we often need to agree upon how particular types of data are encoded into JSON. This is described in the separate **Common Schemas** document.

## Protocol
The **[RGB LN Node](https://github.com/RGB-Tools/rgb-lightning-node/blob/master/openapi.yaml)** currently uses **REST JSON APIs**. 

For consistency, the LSP specification will also use **REST JSON APIs** for the time being.
