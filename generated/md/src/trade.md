---
title: Trade API
pageTitle: Cube - Trade API
description: Trade crypto at microsecond speeds so your market making code never misses a tick.
---

## trade.proto
This schema defines the Protobuf messages used for communication with the
Cube Order Service (Osmium, OS).

### Connection

The order service exposes a websocket endpoint for clients to connect to.
Once connected, clients should submit a [`Credentials`](#credentials)
message, listen for [`Bootstrap`](#bootstrap) messages for resting orders
and positions, and then can begin submitting
[`OrderRequest`](#order-request) and processing
[`OrderResponse`](#order-response).

### Heartbeats

Application-level heartbeats are expected every 30 seconds. If more than one
interval is missed, the order service will disconnect the websocket.

### Price, Quantity, and Lots

All orders are placed on a single market, specified by the market-id. The
market definition specifies the base and quote assets and their respective
lot sizes for the particular market. Prices and quantities in this API are in
units of base and quote _lots_. That is, a quantity of 1 equals 1 base lot,
and a price of 10 equals 10 quote lots / base lot (read as quote lots per
base lot).

For example, consider an ETHBTC market. ETH is the base asset and BTC is the
quote asset. ETH has 18 decimal places (`1 ETH = 10^18 WEI`) and BTC has 8
decimal places (`1 BTC = 10^8 SAT`). Suppose that in this example, the ETHBTC
market has a base lot size of `10^15` and a quote lot size of `10^0` (`1`).
Then an order placed with `quantity = 230` and `price = 6300` in
market-agnostic terms is an order for `0.23 ETH` at a price of `0.06300 BTC /
ETH`. In more detail, we have:

```text
230 base lots
  * (10^15 WEI / base lot)
  / (10^18 WEI / ETH)
  = 230 ETH

6300 quote lots / base lot
  * (1 SAT / quote lot)
  / (10^15 WEI / base lot)
  * (10^18 WEI / ETH)
  / (10^8 SAT / BTC)
  = 0.06300 BTC / ETH
```

### Exchange Order ID

Each order is assigned a unique ID by the exchange. This order ID is
consistent across modifies (including cancel-replace), and other operations.
The exchange order ID can be used to find a particular order in the
market-by-order market data feed, which allows the determination of FIFO
queue priority, etc.

### Transact Time

The transact time is the matching engine timestamp for when an event is
processed. Events that occur with the same transact time occur atomically
from the perspective of the matching engine.



## Credentials
Sent by client on websocket initialization. Once the websocket has been
connected, the client is expected to send this credentials message
immediately. The API key (UUID) and secret access key (hex-encoded 32-byte
array) should be generated on the settings page with the write access. The
signature should be calculated as the concatenation of the byte string
`cube.xyz` and the current unix epoch in seconds interpreted at a
little-endian 64-bit number. For example:

```rust compile_fail
use hmac::{Hmac, Mac, NewMac};
use std::time::SystemTime;

let secret_key = [...];

let timestamp = SystemTime::now()
    .duration_since(SystemTime::UNIX_EPOCH)
    .expect("clock went backwords")
    .as_secs();

let mut mac = Hmac::<sha2::Sha256>::new_from_slice(
    secret_key
).expect("new HMAC error");
mac.update(b"cube.xyz");
mac.update(&timestamp.to_le_bytes());
let signature = <[u8; 32]>::from(mac.finalize().into_bytes());
```


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| access_key_id | [string](#string) |  | Public API key |
| signature | [string](#string) |  | HMAC signature |
| timestamp | [uint64](#uint64) |  | Timestamp in seconds |







## OrderRequest
Every client message, aside from Credentials, must be wrapped as an
OrderRequest.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| new | [NewOrder](#new-order) |  |  |
| cancel | [CancelOrder](#cancel-order) |  |  |
| modify | [ModifyOrder](#modify-order) |  |  |
| heartbeat | [Heartbeat](#heartbeat) |  |  |
| mc | [MassCancel](#mass-cancel) |  |  |







### NewOrder
Place a new order.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| client_order_id | [uint64](#uint64) |  | A unique order ID assigned by the client for this order. The ID must be unique among open orders by this subaccount. |
| request_id | [uint64](#uint64) |  | A request ID that is echoed back on the NewOrderAck or NewOrderReject |
| market_id | [uint64](#uint64) |  |  |
| price | [uint64](#uint64) | optional |  |
| quantity | [uint64](#uint64) |  |  |
| side | [Side](#side) |  |  |
| time_in_force | [TimeInForce](#time-in-force) |  |  |
| order_type | [OrderType](#order-type) |  |  |
| subaccount_id | [uint64](#uint64) |  | The subaccount to place this order on. This subaccount must be writable by the API key specified in the Credentials message. |
| self_trade_prevention | [SelfTradePrevention](#self-trade-prevention) | optional |  |
| post_only | [PostOnly](#post-only) |  |  |







### CancelOrder
Cancel a resting order.
Note that this can be done before the order is acknowledged (an aggressive
cancel) since the identifying field is the `client_order_id`.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| market_id | [uint64](#uint64) |  |  |
| client_order_id | [uint64](#uint64) |  | The order ID specified by the client on the NewOrder request. |
| request_id | [uint64](#uint64) |  | A request ID that is echoed back on the CancelOrderAck or CancelOrderReject |
| subaccount_id | [uint64](#uint64) |  | The subaccount that the NewOrder was placed on. |







### ModifyOrder
Modify a resting order.
- If the `newPrice` and the current resting order's price is the same, and
`newQuantity` is not greater, then the modify is considered a modify down,
and the FIFO queue priority is maintained. Otherwise, the modify-order
request is treated as an atomic cancel-replace and the replacement order is
placed at the end of the FIFO queue for the new price level.
- If post-only is specified and the replacement order would trade, then the
request is rejected and the current resting order remains resting.

Currently, in-flight-mitigation (IFM) is always enabled. That is, the
cumulative fill qty is subtracted from `newQuantity` to calculate the new
resting quantity. For example:

```text
         | Resting | Filled
---------+---------+--------
New 5    | 5       | 0
Fill 2   | 3       | 2
Modify 4 | 2       | 2
```

The post-modify quantity will be `newQuantity - filled = 4 - 2 = 2`.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| market_id | [uint64](#uint64) |  |  |
| client_order_id | [uint64](#uint64) |  | The order ID specified by the client on the NewOrder request. |
| request_id | [uint64](#uint64) |  | A request ID that is echoed back on the ModifyOrderAck or ModifyOrderReject |
| new_price | [uint64](#uint64) |  |  |
| new_quantity | [uint64](#uint64) |  |  |
| subaccount_id | [uint64](#uint64) |  | The subaccount that the NewOrder was placed on. |
| self_trade_prevention | [SelfTradePrevention](#self-trade-prevention) | optional |  |
| post_only | [PostOnly](#post-only) |  |  |







### MassCancel
Cancel all resting orders, optionally limiting to a particular market and /
or order book side.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| subaccount_id | [uint64](#uint64) |  | The subaccount to cancel orders for. |
| request_id | [uint64](#uint64) |  | A request ID that is echoed back on the MassCancelAck and individual CancelOrderAck's. |
| market_id | [uint64](#uint64) | optional | If specified, only orders on the corresponding market will be canceled. |
| side | [Side](#side) | optional | If specified, only orders with this side will be canceled. |







### Heartbeat
A client and server heartbeat. The heartbeat reply, including the timestamp
value, comes from the order service and not the matching engine. Matching
engine timestamps can be extracted from `transact_time` (below).

Latency can be estimated from this, but only the relative difference between
successive server messages should be used. In particular, the client and
server clock should not be expected to be synchronized.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| request_id | [uint64](#uint64) |  | A request ID that is echoed back on the Heartbeat |
| timestamp | [uint64](#uint64) |  |  |







## OrderResponse
Every exchange message after the initial bootstrap will be wrapped as an
OrderResponse.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| new_ack | [NewOrderAck](#new-order-ack) |  |  |
| cancel_ack | [CancelOrderAck](#cancel-order-ack) |  |  |
| modify_ack | [ModifyOrderAck](#modify-order-ack) |  |  |
| new_reject | [NewOrderReject](#new-order-reject) |  |  |
| cancel_reject | [CancelOrderReject](#cancel-order-reject) |  |  |
| modify_reject | [ModifyOrderReject](#modify-order-reject) |  |  |
| fill | [Fill](#fill) |  |  |
| heartbeat | [Heartbeat](#heartbeat) |  |  |
| position | [AssetPosition](#asset-position) |  |  |
| mass_cancel_ack | [MassCancelAck](#mass-cancel-ack) |  |  |







### NewOrderAck
New-order-ack confirms a new-order request. The ack will be ordered before
any fills for this order.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| msg_seq_num | [uint64](#uint64) |  |  |
| client_order_id | [uint64](#uint64) |  | The client order ID specified in the new-order request. |
| request_id | [uint64](#uint64) |  | The request ID specified in the new-order request. |
| exchange_order_id | [uint64](#uint64) |  | [Exchange order ID](#exchange-order-id) |
| market_id | [uint64](#uint64) |  |  |
| price | [uint64](#uint64) | optional | If the order ultimately rests, the `price` field will include the resting price. |
| quantity | [uint64](#uint64) |  | The quantity submitted in the new-order request. |
| side | [Side](#side) |  |  |
| time_in_force | [TimeInForce](#time-in-force) |  |  |
| order_type | [OrderType](#order-type) |  |  |
| transact_time | [uint64](#uint64) |  | [Transact time](#transact-time) |
| subaccount_id | [uint64](#uint64) |  |  |







### CancelOrderAck
Cancel-order-ack confirms a cancel request, or that an order has been
canceled as the result of a different user-initiated reason.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| msg_seq_num | [uint64](#uint64) |  |  |
| client_order_id | [uint64](#uint64) |  |  |
| request_id | [uint64](#uint64) |  | If the Reason is `DISCONNECT`, `IOC`, `STP_RESTING`, or `STP_AGGRESSING`, this request ID will be `u64::MAX`. Otherwise, it will be the request ID of the initiated cancel action. For a mass cancel, each cancel order ack will have the MassCancel's request_id. |
| transact_time | [uint64](#uint64) |  | [Transact time](#transact-time) |
| subaccount_id | [uint64](#uint64) |  |  |
| reason | [CancelOrderAck.Reason](#cancel-order-ack-reason) |  |  |
| market_id | [uint64](#uint64) |  |  |







### ModifyOrderAck
Modify-order-ack confirms a modify-order request. If the modify resulted in
an aggressing cancel-replace, the ack will be ordered before any fills for
this order.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| msg_seq_num | [uint64](#uint64) |  |  |
| client_order_id | [uint64](#uint64) |  |  |
| request_id | [uint64](#uint64) |  | The request ID specified in the modify request. |
| transact_time | [uint64](#uint64) |  | [Transact time](#transact-time) |
| new_quantity | [uint64](#uint64) |  | The quantity submitted in the modify request after applying IFM logic. |
| subaccount_id | [uint64](#uint64) |  |  |
| market_id | [uint64](#uint64) |  |  |
| price | [uint64](#uint64) |  |  |







### MassCancelAck
Mass-cancel-ack confirms a mass-cancel request. If `reason` is set, the mass
cancel was not applied and there are no affected orders. Individual
CancelOrderAck's will be sent for each order that was affected.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| msg_seq_num | [uint64](#uint64) |  |  |
| subaccount_id | [uint64](#uint64) |  |  |
| request_id | [uint64](#uint64) |  | The request ID specified in the mass-cancel request. |
| transact_time | [uint64](#uint64) |  | [Transact time](#transact-time) |
| reason | [MassCancelAck.Reason](#mass-cancel-ack-reason) | optional |  |
| total_affected_orders | [uint32](#uint32) |  | The total number of orders that were canceled. |







### NewOrderReject
New-order-reject indicates that a new-order request was not applied.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| msg_seq_num | [uint64](#uint64) |  |  |
| client_order_id | [uint64](#uint64) |  | The client order ID specified in the new-order request. |
| request_id | [uint64](#uint64) |  | The request ID specified in the new-order request. |
| transact_time | [uint64](#uint64) |  | [Transact time](#transact-time) |
| subaccount_id | [uint64](#uint64) |  |  |
| reason | [NewOrderReject.Reason](#new-order-reject-reason) |  |  |
| market_id | [uint64](#uint64) |  |  |
| price | [uint64](#uint64) | optional |  |
| quantity | [uint64](#uint64) |  |  |
| side | [Side](#side) |  |  |
| time_in_force | [TimeInForce](#time-in-force) |  |  |
| order_type | [OrderType](#order-type) |  |  |







### CancelOrderReject
Cancel-order-reject indicates that a cancel-order request was not applied.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| msg_seq_num | [uint64](#uint64) |  |  |
| client_order_id | [uint64](#uint64) |  | The client order ID specified in the cancel-order request. |
| request_id | [uint64](#uint64) |  | The request ID specified in the cancel-order request. |
| transact_time | [uint64](#uint64) |  | [Transact time](#transact-time) |
| subaccount_id | [uint64](#uint64) |  |  |
| reason | [CancelOrderReject.Reason](#cancel-order-reject-reason) |  |  |
| market_id | [uint64](#uint64) |  |  |







### ModifyOrderReject
Modify-order-reject indicates that a modify-order request was not applied.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| msg_seq_num | [uint64](#uint64) |  |  |
| client_order_id | [uint64](#uint64) |  | The client order ID specified in the modify-order request. |
| request_id | [uint64](#uint64) |  | The request ID specified in the modify-order request. |
| transact_time | [uint64](#uint64) |  | [Transact time](#transact-time) |
| subaccount_id | [uint64](#uint64) |  |  |
| reason | [ModifyOrderReject.Reason](#modify-order-reject-reason) |  |  |
| market_id | [uint64](#uint64) |  |  |







### Fill
A fill for an order.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| msg_seq_num | [uint64](#uint64) |  |  |
| market_id | [uint64](#uint64) |  |  |
| client_order_id | [uint64](#uint64) |  | The client order ID specified in the new-order request. |
| exchange_order_id | [uint64](#uint64) |  | [Exchange order ID](#exchange-order-id) |
| fill_price | [uint64](#uint64) |  | The price that this trade occured at. |
| fill_quantity | [uint64](#uint64) |  | The quantity that was traded in this fill. |
| leaves_quantity | [uint64](#uint64) |  | The remaining quantity for this order after the fill is applied. |
| transact_time | [uint64](#uint64) |  | [Transact time](#transact-time) |
| subaccount_id | [uint64](#uint64) |  |  |







### AssetPosition
The user's underlying asset position. These are sent asynchronously as
positions are updated and broadcast through internal position channels. They
can also be tracked by applying other OrderResponse messages individually.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| subaccount_id | [uint64](#uint64) |  |  |
| asset_id | [uint64](#uint64) |  |  |
| total | [RawUnits](#raw-units) |  |  |
| available | [RawUnits](#raw-units) |  | The available amount after open orders are subtracted. |







### RawUnits
Raw-units is a 256-bit number for the amount of an asset. The precision is
based on the underlying asset. For example, ETH is specified as if in
fixed-point 10^18, while BTC is specified as if in fixed-point 10^8.

The number is interpreted in 'little-endian' as `[word0, word1, word2,
word3]`.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| word0 | [uint64](#uint64) |  |  |
| word1 | [uint64](#uint64) |  |  |
| word2 | [uint64](#uint64) |  |  |
| word3 | [uint64](#uint64) |  |  |







## Bootstrap
A bootstrap message sent after Credentials authentication.
Client resting and pending orders used to bootstrap state. Sent as the first
message(s) after initialization. Bootstrap is complete after a message tagged
`Done` is received and every message after that will be an `OrderResponse`.
Multiple messages may be received for `RestingOrders` and `AssetPositions`
and these should be concatenated.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| done | [Done](#done) |  |  |
| resting | [RestingOrders](#resting-orders) |  |  |
| position | [AssetPositions](#asset-positions) |  |  |







### RestingOrders
A chunk of resting orders. Sent on bootstrap.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| orders | [RestingOrder](#resting-order) | repeated |  |







### AssetPositions
A chunk of asset positions. Sent on bootstrap.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| positions | [AssetPosition](#asset-position) | repeated |  |







### Done
An indication that bootstrap is complete.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| latest_transact_time | [uint64](#uint64) |  | [Transact time](#transact-time) |







### RestingOrder
A resting order. Sent on bootstrap in `RestingOrders`.


| Field | Type | Label | Description |
| ----- | ---- | ----- | ----------- |
| client_order_id | [uint64](#uint64) |  | The client order ID specified in the new-order request. |
| exchange_order_id | [uint64](#uint64) |  | [Exchange order ID](#exchange-order-id) |
| market_id | [uint64](#uint64) |  |  |
| price | [uint64](#uint64) |  |  |
| original_quantity | [uint64](#uint64) |  | The quantity submitted in the new-order request. |
| side | [Side](#side) |  |  |
| time_in_force | [TimeInForce](#time-in-force) |  |  |
| order_type | [OrderType](#order-type) |  |  |
| remaining_quantity | [uint64](#uint64) |  | The current remaining quantity on the book. |
| rest_time | [uint64](#uint64) |  | [Transact time](#transact-time) of the NewOrderAck |
| subaccount_id | [uint64](#uint64) |  |  |







## Enums



### Side
Side specifies whether the order is buying or selling the base asset. A trade
is matched when a buyer (BID) and a seller (ASK) agree on a price (cross).
The bid-ask spread is the gap between the highest bid price and lowest ask
price on the orderbook.

| Name | Number | Description |
| ---- | ------ | ----------- |
| BID | 0 | A bid order buys the base asset with the quote asset. |
| ASK | 1 | An ask (or offer) order sells the base asset and gets the quote asset. |




### TimeInForce
Time-in-force (TIF) specifies how long the order remains in effect.

| Name | Number | Description |
| ---- | ------ | ----------- |
| IMMEDIATE_OR_CANCEL | 0 | Immediate-or-cancel (IOC), also known as fill-and-kill (FAK), orders are immediately executed against resting orders. If the order cannot be fully filled, the remaining balance will be canceled, and an additional CancelOrderAck with the IOC reason will be sent. |
| GOOD_FOR_SESSION | 1 | Good-for-session (GFS) orders are active until they are completely executed, canceled, or when the session expires. |
| FILL_OR_KILL | 2 | Fill-or-kill (FOK), also known as all-or-none (AON), orders must be filled immediately against resting orders or the entire order is canceled. |




### OrderType
Order-type specifies how the order will be placed into the order book.

| Name | Number | Description |
| ---- | ------ | ----------- |
| LIMIT | 0 | A limit order is accompanied with a price (inclusive) that specifies the upper limit to buy and the lower limit to sell. If the price is not immediately available and the TIF allows resting orders, the limit order will rest until filled or canceled. |
| MARKET_LIMIT | 1 | A market limit order crosses the bid-ask spread and, if not fully filled, becomes a limit order at the best available market price. If there is no opposing market, the order is rejected with the NO_OPPOSING_LIMIT_ORDER reason. Price must be null. |
| MARKET_WITH_PROTECTION | 2 | A market with protection order crosses the bid-ask spread and continues to cross until the order is fully filled or the price protection level, defined by the best market price widened by a market-specific protection point count, is reached. If there is no opposing market, the order is rejected with the NO_OPPOSING_LIMIT_ORDER reason. Price must be null. |




### SelfTradePrevention
Self-trade-prevention (STP) allows market participants to prevent the matching
of orders for accounts with common ownership. Currently, STP only applies for
orders with the same subaccount_id. STP will only be applied when a match is
about to occur between the two orders. That is, if the aggressing order is
fully filled before reaching the resting order in FIFO order, no STP cancels
will happen.

| Name | Number | Description |
| ---- | ------ | ----------- |
| CANCEL_RESTING | 0 | Cancel-resting specifies that if a self-trade is about to occur, the resting order should be canceled instead and further order book processing should occur as normal. |
| CANCEL_AGGRESSING | 1 | Cancel-aggressing specifies that if a self-trade is about to occur, the aggressing order should be canceled instead and no further action should be taken. |
| ALLOW_SELF_TRADE | 2 | Allow-self-trade disables STP functionality. |




### PostOnly
Post-only specifies whether a new order is allowed to immediately execute.
Post-only cannot be enabled with market orders or with a TIF that does not
allow resting orders.

| Name | Number | Description |
| ---- | ------ | ----------- |
| DISABLED | 0 |  |
| ENABLED | 1 |  |




### CancelOrderAck.Reason


| Name | Number | Description |
| ---- | ------ | ----------- |
| UNCLASSIFIED | 0 |  |
| DISCONNECT | 1 |  |
| REQUESTED | 2 | This order was specified in a cancel request. |
| IOC | 3 | This was an IOC new-order that does not get fully filled. |
| STP_RESTING | 4 | A resting order was STP canceled. |
| STP_AGGRESSING | 5 | An aggressing order was STP canceled. |
| MASS_CANCEL | 6 | This order was covered by a mass-cancel request. |




### MassCancelAck.Reason


| Name | Number | Description |
| ---- | ------ | ----------- |
| UNCLASSIFIED | 0 |  |
| INVALID_MARKET_ID | 1 |  |
| INVALID_SIDE | 2 |  |




### NewOrderReject.Reason
Reasons that are prefixed with `INVALID_` normally indicate that the
corresponding field did not take a valid value.

| Name | Number | Description |
| ---- | ------ | ----------- |
| UNCLASSIFIED | 0 |  |
| INVALID_QUANTITY | 1 | Quantity was zero. |
| INVALID_MARKET_ID | 2 | The specified market ID does not exist. |
| DUPLICATE_ORDER_ID | 3 | The specified client order ID was not unique among open orders for this subaccount. |
| INVALID_SIDE | 4 |  |
| INVALID_TIME_IN_FORCE | 5 |  |
| INVALID_ORDER_TYPE | 6 |  |
| INVALID_POST_ONLY | 7 |  |
| INVALID_SELF_TRADE_PREVENTION | 8 |  |
| UNKNOWN_TRADER | 9 | Internal error: the matching engine could not find this subaccounts positions. |
| PRICE_WITH_MARKET_ORDER | 10 |  |
| POST_ONLY_WITH_MARKET_ORDER | 11 |  |
| POST_ONLY_WITH_INVALID_TIF | 12 |  |
| EXCEEDED_SPOT_POSITION | 13 | The sum of open orders and this new-order would exceed the subaccounts spot limits. |
| NO_OPPOSING_LIMIT_ORDER | 14 |  |
| POST_ONLY_WOULD_TRADE | 15 | The post-only order would have crossed and traded. |
| DID_NOT_FULLY_FILL | 16 | A FOK was not fully fillable against resting orders at the requested price and quantity. |




### CancelOrderReject.Reason


| Name | Number | Description |
| ---- | ------ | ----------- |
| UNCLASSIFIED | 0 |  |
| INVALID_MARKET_ID | 1 | The specified market ID does not exist. |
| ORDER_NOT_FOUND | 2 | The specified client order ID does not exist for the corresponding market ID and subaccount ID. |




### ModifyOrderReject.Reason
Reasons that are prefixed with `INVALID_` normally indicate that the
corresponding field did not take a valid value.

| Name | Number | Description |
| ---- | ------ | ----------- |
| UNCLASSIFIED | 0 |  |
| INVALID_QUANTITY | 1 | Quantity was zero. |
| INVALID_MARKET_ID | 2 | The specified market ID does not exist. |
| ORDER_NOT_FOUND | 3 | The specified client order ID does not exist for the corresponding market ID and subaccount ID. |
| INVALID_IFM | 4 |  |
| INVALID_POST_ONLY | 5 |  |
| INVALID_SELF_TRADE_PREVENTION | 6 |  |
| UNKNOWN_TRADER | 7 | Internal error: the matching engine could not find this subaccounts positions. |
| EXCEEDED_SPOT_POSITION | 8 | If the modify-order would cause a cancel-replace, the sum of open orders and this replacement order would exceed the subaccounts spot limits. |
| POST_ONLY_WOULD_TRADE | 9 | If the modify-order would cause a cancel-replace, the post-only replacement would have crossed and traded. |








## Scalar Value Types

| .proto Type | Notes | Rust | C++ | Python | Go |
| ----------- | ----- | ---- | --- | ------ | -- |
| double |  | f64 | double | float | float64 |
| float |  | f32 | float | float | float32 |
| int32 | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead. | i32 | int32 | int | int32 |
| int64 | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead. | i64 | int64 | int/long | int64 |
| uint32 | Uses variable-length encoding. | u32 | uint32 | int/long | uint32 |
| uint64 | Uses variable-length encoding. | u64 | uint64 | int/long | uint64 |
| sint32 | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s. | i32 | int32 | int | int32 |
| sint64 | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s. | i64 | int64 | int/long | int64 |
| fixed32 | Always four bytes. More efficient than uint32 if values are often greater than 2^28. | u64 | uint32 | int | uint32 |
| fixed64 | Always eight bytes. More efficient than uint64 if values are often greater than 2^56. | u64 | uint64 | int/long | uint64 |
| sfixed32 | Always four bytes. | i32 | int32 | int | int32 |
| sfixed64 | Always eight bytes. | i64 | int64 | int/long | int64 |
| bool |  | bool | bool | boolean | bool |
| string | A string must always contain UTF-8 encoded or 7-bit ASCII text. | String | string | str/unicode | string |
| bytes | May contain any arbitrary sequence of bytes. | Vec<u8> | string | str | []byte |

