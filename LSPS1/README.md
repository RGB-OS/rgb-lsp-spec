# RGB LSPS1 RGB Denominated Channel Request

| Name    | `channel_request` |
|---------|------------------|
| Version | 0 |
| Status  | Draft |

## Motivation

The goal of this specification is to provide a standardized RGB LSP API for wallets to purchase an RGB Denominated channel from the LSP directly.

## Definitions

### Data types

This specification uses data types defined in [LSPS0 Common Schemas][LSPS0.common_schemas].

### Actors

`LSP` is the API provider. `Client` is the client of the API.

### Channel sides

`client_balance` are the funds on the client side. `lsp_balance` are the funds on the LSP side.

### Overprovisioning

The LSP is allowed to overprovision channels/on-chain-payments/on-chain-fees as long as it benefits the client.

> **Rationale** The LSP may need to "bin" UTXOs.

### Errors

LSP will return errors according to the [JSON-RPC 2.0](https://www.jsonrpc.org/specification) specification (see [LSPS0 Error Handling](https://github.com/BitcoinAndLightningLayerSpecs/lsp/tree/main/LSPS0#error-handling)).

## Trust model

In general, LSPS1 is developed on the basis that the client trusts the LSP to deliver the promised goods. Channel purchases are not atomic and therefore the client risks not getting the promised goods if the LSP is malicious.

## Order Flow Overview

* Client calls `lsps1.get_info` to get the LSP's options.
* Client calls `lsps1.create_order` to create an order.
* Client pays the order either on-chain or off-chain.
* LSP opens the channel as soon as the payment is confirmed.
  * Channel open failed: LSP refunds the client.

## API

### 1. lsps1.get_info

| JSON-RPC Method | lsps1.get_info |
|---------------- |---------------|
| Idempotent      | Yes |

`lsps1.get_info` is the entry point for each client using the API. It lists all options in a dictionary.

- The LSP SHOULD NOT change the values in `lsps1.get_info` more than once per day.

> **Rationale Change frequency** The LSP should not change values in `lsps1.get_info` too frequently. Lightning Explorers may scrape these values and provide an overview of all LSPs. If the values change too frequently, Lightning Explorers may not be able to keep up with the changes. Changing them a maximum of once a day gives explorers enough time to scrape. Once a day has been chosen as it is a similar rate-limit that core-lightning puts on the lightning gossip.

The client MUST call `lsps1.get_info` first.

**Request** No parameters needed.

**Response**

```json
{
  "options": {
      "min_required_channel_confirmations": 0,
      "min_funding_confirms_within_blocks": 6,
      "min_onchain_payment_confirmations": null,
      "supports_zero_channel_reserve": true,
      "min_onchain_payment_size_sat": null,
      "max_channel_expiry_blocks": 20160,
      "min_initial_client_balance_sat": "20000",
      "max_initial_client_balance_sat": "100000000",
      "min_initial_lsp_balance_sat": "0",
      "max_initial_lsp_balance_sat": "100000000",
      "min_channel_balance_sat": "50000",
      "max_channel_balance_sat": "100000000"
  }
}
```

### 2. lsps1.create_order

| JSON-RPC Method     | lsps1.create_order |
|-------------------- |------------------- |
| Idempotent          | No                 |

**Request**

```json
{
  "lsp_balance_sat": "5000000",
  "client_balance_sat": "2000000",
  "required_channel_confirmations": 0,
  "funding_confirms_within_blocks": 6,
  "channel_expiry_blocks": 144,
  "token": "",
  "refund_onchain_address": "bc1qvmsy0f3yyes6z9jvddk8xqwznndmdwapvrc0xrmhd3vqj5rhdrrq6hz49h",
  "announce_channel": true
}
```

### 3. lsps1.get_order

| JSON-RPC Method | lsps1.get_order |
|---------------- |---------------- |
| Idempotent      | Yes             |

**Request**

```json
{
  "order_id": "bb4b5d0a-8334-49d8-9463-90a6d413af7c"
}
```

**Response** is the same as defined in `lsps1.create_order`.

[LSPS0.common_schemas]: ../LSPS0/common-schemas.md
