# Zilliqa Mining Protocol ("ZMP") v1

# Abstract

Previously, mining Zilliqa PoW rounds was merged with Ethereum Stratum Mining Protocol. As Ethereum goes Proof-of-Stake and no longer becomes the #1 PoW coin, there is an increasing need for an alternative mining protocol compatible with mining Zilliqa in dual with other non-Ethash coins.

*Ethereum Classic Compatibility Note: Up until Zilliqa Mainnet's DAG epoch is not 0, Zilliqa can still be mined while being merged with Ethereum Classic Stratum Mining Protocol, as Etchash and Ethash DAGs on Epoch 0 are the same.*

## Specification Conventions

The keywords `MUST`, `MUST NOT`, `REQUIRED`, `SHALL`, `SHALL NOT`, `SHOULD`, `SHOULD NOT`, `RECOMMENDED`, `MAY`, and `OPTIONAL` in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The words "miner", "user", and "client" should be considered as a mining rig that is connecting to the mining pool, referenced as "pool" and "server".

# Specification

Similar to standard Stratum Mining Protocols, ZMP relies on a basic TCP connection and JSON RPC 2.0-style commands with ID fields.

### JSON RPC 2.0 Compliances
- Each Request and Response MUST contain `id` field.
- The Response `id` field MUST match the Request `id` field.
- Notifications MUST NOT have an `id` field. Any incoming/outgoing message without the `id` field MUST be considered as a notification.

### JSON RPC 2.0 Differences
- `jsonrpc` field indicating the JSON RPC version MUST always be omitted.
- `id` field MUST be a uint32 integer. The server MUST reject all messages with non-integer `id` fields that exceed the uint32 bounds, which are [0, 2^32).
- `result` field CAN be missing in Responses, indicating that the Request was processed successfully without errors and additional information to be provided to the client.
- Notifications CAN contain an `error` field.

## Protocol Conventions
- A byte array or number field MUST always be encoded in hex, WITHOUT the `0x` prefix. 
- The unknown Object keys MUST be ignored if Object is used as `params` or `result`. This is made to allow for future protocol upgrades/extensions.
- In case the server received a request with unknown RPC method, the server MUST NOT terminate the connection.

### Request

A Request is a JSON Object that has the following fields:
- Required `id` - Specifies the Request ID.
- Required `method` - Specifies the RPC call method.
- Optional `params` - This field MUST be an array. It specifies the arguments to the RPC call. The client SHOULD NOT specify an empty array (`{"params": []}`). The client SHOULD omit this field altogether if there are no parameters to pass.

### Response
- Required `id` - MUST be the same as specified in the Request.
- Optional `result` - MUST NOT be set to true. Instead, this field should be omitted altogether. Specifies the result of the call execution. This field CAN be any JSON type.
- Optional `error` - MUST always be a string. MUST NOT coexist with the `result` field. Specifies the error. If there are no errors, this field MUST be omitted.

### Notification
- Required `result` - Specifies the notification payload. As with Response, this field CAN be any JSON type.
- Optional `error` - Specifies a global session-wide error. This field MUST NOT coexist with `result`. The miner MUST expect the global error to occur at any time.

Note: `id` field **MUST NOT BE PRESENT IN THE NOTIFICATION**.

### Keepalive Message

The keepalive message is an empty JSON Object (i.e., `{}`).

### Boolean Responses
Boolean Responses like `{"result": true}` or `{"result": false}` MUST not be sent.

Instead of specifying `{"result": true}`, the server MUST omit the `result` field altogether, leaving a JSON message with `id` field only.
```js
{"id": 5}
```

Instead of specifying `{"result": false}`, the server MUST provide the `error` field with a description of the error instead.
```js
{"id": 5, "error": "Incorrect Solution"}
```

## Connection
Upon establishing a TCP connection to the server, the client MUST authenticate using the `login` method.

Any client Requests that are not `login` method MUST be rejected if no login credentials were provided.

The server MUST NOT terminate the connection if the client tries to execute a non-login command when no login credentials are provided.

## Authentication
At the start of the session, the client MUST authenticate, providing the login credentials.

The authentication Request's method is `login`. A single JSON Object MUST be passed in an array as `params`, with the Object having these fields:
- Required `userAgent` - Specifies the mining software name and version, MUST be following the `NAME/VERSION` scheme.
- Required `login` - Specifies the login string. It is up to the pool to decide the format of this field. It is RECOMMENDED to use `WALLET_ADDRESS.WORKER_NAME` scheme.
- Optional `password` - Specifies the password, useful for private proxies.

If the authentication is successful, the server MUST reply with an object containing these fields:
- Required `epoch` - The current Zilliqa DS epoch. This is needed for the miner to determine the current DAG epoch.

Example of the authentication message:
```js
// client -> server
{"id": 0, "method": "login", "params": [{"userAgent": "ExampleMiner/v1.0.0", "login": "wallet_address.worker_name"}]}
```

The server MUST reply with a Response. As mentioned in the Boolean Responses section, the Response without `result` and `error` should be considered as successful.
```js
// server -> client
{"id": 0, "result": {"epoch": "57b9"}}
```

In case of an error, the server MUST reply with a Response containing an `error` field, as defined in the Response specification.
```js
// server -> client
{"id": 0, "error": "Invalid Login Credentials"}
```

Note: this specification does not define the error strings. It is up to the pool to choose them.

The server MUST NOT terminate the connection if the login credentials are incorrect.

## Work Notification

When the Zilliqa Proof-of-Work round starts, the server MUST send a notification with the job information.

The `result` MUST be either `null` or an array with a single Object containing the following fields:
- Required `sealHash` - aka header hash, the Proof-of-Work hash to be passed to the PoW hash function. 
- Required `diff` - hex-encoded uint256 (without leading zeros) difficulty of the Proof-of-Work job. It is an inverse target (aka boundary), calculated by `2^256 / <target>`.
- Required `epoch` - hex-encoded uint64 (without leading zeros) Zilliqa DS Epoch, used as the block number passed to the Ethash function to generate a proper DAG.
- Required `expires` - hex-encoded uint64 UNIX timestamp in milliseconds specifying when the job expires. The client MUST stop mining the job after it expires, as the server MUST NOT accept solutions that are too late.
- Required `ttl` - hex-encoded uint64 (without leading zeroes) time to live in milliseconds for this work. This is created as an alternative for the `expires` field, given that the client's clock may be off. The client MUST stop mining whichever hits first, `now+ttl`, or `expires`. It is RECOMMENDED for the client to verify that the system time is set correctly by checking that `expires` and `now+ttl` are the same +/- communication latency.

It is RECOMMENDED for the client to check that the PoW duration does not exceed 120 seconds, and if so, reject mining the job as no Zilliqa job is supposed to take more than 60 seconds.

If the `result` sent by the server is `null`, the client MUST stop mining immediately. `null` result serves as a cancelation flag, meaning that no further work is needed. The server might send multiple work notifications during a single PoW round, the client MUST switch to new jobs immediately as no rewards are given for stale shares.

Example:
```js
// server -> client
{"result": {"sealHash": "3d2dcbf8dedab8f0404b0875d046ce85b272cf377d4b6f1a10137c9517b6417f", "diff": "ee6b2800", "epoch": "57b9", "ttl": "4e20", "expires": "18272e43e28"}}
```

## Solution/Share Submission
During the Zilliqa Proof-of-Work round, the miner submits the shares by sending a Request with `submit` method.

The `params` field MUST be an array with a single Object containing the following fields:
- Required `n` - Shorthand for "Nonce".
- Optional (**FOR DEBUG ONLY - DO NOT USE THIS IN PRODUCTION**) `sealHash` - Sealhash indication, created for debugging. If specified, the server MUST check if the indicated seal hash matches the current active job.

Example:
```js
// client -> server
{"id": 1, "method": "submit", "params": [{"n": "9a40a6819517f2b2"}]}
// server -> client
{"id": 2}
```


## Keepalives

Due to the temporary nature of Proof-of-Work rounds on Zilliqa, the protocol implies long waiting gaps leaving the connection open without any communication. To make the client/server aware that both of them are still there and that they are not dead but waiting for the next PoW round to start, ZMP includes keepalive messages.

Keepalive messages are originally sent by the server, and then MUST replied by the client in ping-pong manner.

It is RECOMMENDED to use keepalives to avoid dangling connections. The server SHOULD send keepalive messages with a 60-second interval.

Example:
```js
// server -> client
{}
// client -> server
{}
```

If the server received no keepalive replies after 120 seconds since the last keepalive message sent by the user, the server SHOULD terminate the connection upon sending the Notification with the error:
```js
// server -> client
{"error": "No keepalives received after 120 seconds since the last keepalive message"}
```

## Connection termination

The connection is terminated by closing the underlying TCP connection. 

## Default Ports

| Type | Port |
| ---- | ---- |
| TCP  | 9486 |
| SSL  | 9487 |

In production, the server MUST use SSL ports by default.


## Connection string

Connection string should use a `zmp://` (and `zmp+ssl://` for consistency) prefix for SSL, and `zmp+tcp://` for TCP followed by host and optionally port (if it is not the default one). Example: `zmp://zil.flexpool.io`.

# Example Communication

The client establishes a TCP connection to the server. After that, the client authenticates with this message:
```js
// client -> server
{"id": 0, "method": "login", "params": [{"userAgent": "ExampleMiner/v1.0.0", "login": "zil1dsj277fa6kl4zskvpr2298l9ye0g90rsnvcms7.worker_name"}]}
```

If provided credentials are valid, the server replies with the current network DS epoch to be used to compute the DAG in advance, before the PoW round starts:
```js
// server -> client
{"id": 0, "result": {"epoch": "57b9"}}
```

Otherwise, in case there is an error, the server replies with:
```js
// server -> client
{"id": 0, "error": "Invalid Login Credentials"}
```
The actual error may be different. There are no requirements on exact error wording.

Now here, the most boring part happens when the user and server are waiting for the next Zilliqa PoW round to occur (every 2.5 hours on average). Meanwhile, the server and user will send keepalives to each other to ensure both are online. See the Keepalives section for more information.

When the PoW round starts, the server immediately notifies the user with the Notification that looks like this:
```js
// server -> client
{"result": {"sealHash": "3d2dcbf8dedab8f0404b0875d046ce85b272cf377d4b6f1a10137c9517b6417f", "diff": "ee6b2800", "epoch": "57b9", "ttl": "4e20", "expires": "18272e43e28"}}
```

The client starts mining until the `expires` timestamp hits, or `now+ttl`, whichever is the first. See the Work Notification section for more information.

If the client found a solution for a provided PoW problem, the client submits it:
```js
// client -> server
{"id": 1, "method": "submit", "params": [{"n": "9a40a6819517f2b2"}]}
```

The client provides a single param to the server, specifying the hex-encoded uint64 nonce.

If the solution provided is correct, the server replies with:
```js
// server -> client
{"id": 1}
```

In case the solution is incorrect, the server returns an error:
```js
// server -> client
{"id": 1, "error": "Incorrect Solution"}
```

Similarly, if the job has expired:
```js
// server -> client
{"id": 1, "error": "Job Expired"}
```

If the user wants to terminate the session, the user should close the TCP connection.

# FAQ

## Why JSON?

JSON is the most common data serialization format, included in every programming language. It is also human readable, making the server-side debugging and connection tracing easier.

## Why Objects for parameters instead of plain array fields?
Because using keyed Objects makes things easier for the development and also allows seamless backward-compatible protocol upgrades, as unknown fields are ignored.


# Client Implementation Checklist

- [ ] My client is properly replying to keepalives when requested by server as `{}`. My client does not send keepalives on its own.
- [ ] My client does not specify params in arrays, instead uses a single object with all parameters.
- [ ] My client does not include unnecessary fields like `worker` in `submit` message (commonly found in other mining protocols).
- [ ] My client does not include the sealhash indication when submitting shares when ran in production.
- [ ] My client does not assume that the DAG is fixed at 0, and generates the DAG dynamically based on the current network DS epoch.
- [ ] My client stops mining when it sees a work notification with `{"result": null}` (cancel flag).
- [ ] My client is fully capable of handling temporary disconnections as part of maintenances on the pool side, which do not affect the profit due to the temporary nature of Zilliqa mining.