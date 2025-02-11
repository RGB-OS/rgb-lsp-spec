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

**Response**

```json
{
    "order_id": "66e42dba6afb292692ade4e7",
    "status": "Pending",
    "created_at": "2024-09-13T12:19:06.991Z",
    "updated_at": "2024-09-13T12:19:06.991Z",
    "lsp_balance_sat": 100000,
    "channel_expiry_blocks": 12960,
    "refund_on_chain_address": "02c9d1afd383fb651d93975d2a2e727aca5e6dd4fa2f090fded50102a149969ce8",
    "public_key": "02c9d1afd383fb651d93975d2a2e727aca5e6dd4fa2f090fded50102a149969ce8@3dc0a5a72c2748e0bcd3b6c8fea9db9c.80c65b6183a949d1a642f5dd64a4f5e9.peers.thunderstack.org:12187",
    "payment": {
        "ln": {
            "type": "ln",
            "status": "Pending",
            "fee": 251515,
            "invoice": "lnbcrt2515150n1pn6k2y7dqud3jxktt5w46x7unfv9kz6mn0v3jsnp4qdh55vg4n3pgt2a635zue44w2tzl00ast96ks3jelxkq873zl3ljcpp5ft2zj96v64tya0c08evjq5dg5jxkkfwy4m5mnrg7p4j4tjk0znxssp530hy83x7gr6ldrawk0xe3rphz0uc9v7x8cpfequgnuq493t57uuq9qyysgqcqpcxqrpcgwnwlpmaq2lm24mjsef5sqtycpydl543a00jw2sthjtwgzdshfeunjs20uuj356ks6lhzrutcjyu9x6q38d244aafwpvxqc34jawxyzspxxn6zw",
            "updated_at": "2025-02-11T10:38:22.138Z",
            "created_at": "2025-02-11T10:38:22.138Z",
            "expires_at": "2025-02-11T11:08:22.138Z"
        },
        "btc_onchain": {
            "type": "btc_onchain",
            "status": "Pending",
            "fee": 251515,
            "invoice": "bitcoin:bcrt1q36xg65c3wdj969f92f2wd0za30z3lzq0envx88?amount=0.00251515&label=67ab289d69711ad1f44aa8e6&message=Payment%20for%20order%3A%2067ab289d69711ad1f44aa8e6",
            "updated_at": "2025-02-11T10:38:22.390Z",
            "created_at": "2025-02-11T10:38:22.390Z",
            "expires_at": "2025-02-11T11:38:22.390Z"
        }
    },
    "channel": null
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
