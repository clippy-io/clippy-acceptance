clippy-acceptance
=================

Acceptance test to be run against Clippy API implementations to check for
completeness. This repo also contains documentation useful for creating your own
implementation of clippy.

## Design Goals

1. **Complete federation of clients.** The server there strictly to help pass
   ephemeral messages between clients. Clients can talk to any server that
   passes this acceptance test.
2. **Server never sees plaintext messages.** Messages pass through the server
   GPG encrypted.
3. **Server never sees private keys.** Private keys are generated by the client
   and the server never knows about them.

## Run the tests

### Install

```bash
git clone git@github.com:clippy-io/clippy-acceptance.git
cd clippy-acceptance
npm install
```

### Run

```bash
npm test
```

### Environment Variables

You can manually override the environment specific default by specifying it as
an environment variable. See [node-config][].

| (↓`NODE_ENV`) | `ENDPOINT_BASE`       |
|---------------|-----------------------|
| default       | http://localhost:9001 |
| development   |                       |
| production    |                       |

## Implementation

### Models

#### `Client`<a name="Client"></a>

This should not be stored on the server. This is a contract between the client
and the server.

| Property  | Type                              | Description                                           |
|-----------|-----------------------------------|-------------------------------------------------------|
| id        | String, [UUIDv4][]                | Identifies the individual client                      |
| group     | String, [UUIDv4][]                | Identifies the group of devices the client belongs to |
| name      | String                            | Friendly, identifiable name                           |
| peers     | Array of [Client][] Objects       | All of the clients this client knows about            |
| publicKey | String, [GPG Public Key][rfc4880] | GPG public key                                        |

#### `SyncRequest`<a name="SyncRequest"></a>

This is briefly stored on the server and should have an expiration.

| Property  | Type                  | Description                                                       |
|-----------|-----------------------|-------------------------------------------------------------------|
| group     | String, [UUIDv4][]    | Identifies the group of devices                                   |
| code      | String, [SyncCode][]  | Used to look up sync request                                      |
| status    | String                | Can be any of: `accepted`, `pending`                              |
| initiator | Object, [Client][]    | `Client` that is inviting the target client into the device group |
| target    | Object, [Client][]    | Target `Client` that is being invited into the device group       |
| createdAt | String, UTC Timestamp | Creation time
| expiresAt | String, UTC Timestamp | Future expiration date

#### `Error`<a name="Error"></a>

Error objects MUST be returned as a collection keyed by "errors" in the top
level of a JSON API document, and SHOULD NOT be returned with any primary data.
Inspired by [json api](http://jsonapi.org/format/#errors).

| Property       | Type             | Description                                                                                                                |
|----------------|------------------|----------------------------------------------------------------------------------------------------------------------------|
| message        | String           | A short, human-readable summary of the problem.                                                                            |
| fieldNames     | Array of Strings | If applicable, the problematic field names.                                                                                |
| classification | String           | An application-specific error code. Can be any of `RequiredError`, `ContentTypeError`, `DeserializationError`, `TypeError` |

### REST API

#### `GET /capabilities`

Returns an object that describes the server's capabilities.

```json
{
  "name": "Clippy Server: Golang Edition",
  "version": "1.0.0",
  "capabilities": [
    "REST"
  ]
}
```

#### `POST /sync`

Create a `SyncRequest`. 

##### Request

Initiator Client sends it's personal [Client][] object.

Required fields:

- id
- group 
- name 
- peers
- publicKey

##### Response

Returns a [SyncRequest][] with the following properties defined:

- group
- code
- status: `pending`
- initiator: the [Client][] object of the Client that sent the request

#### `POST /sync/{{ code }}`

Update a Sync Request as a target.

##### Request

The target Client sends it's personal [Client][] object.

Required Fields:

- id
- name
- publickey

##### Response

Returns a [SyncRequest][] with the following properties defined:

- group
- code
- status: `accepted`
- initiator: the [Client][] object of the initiator Client that created the [SyncRequest][] initially.
- target: the [Client][] object of target Client that just sent the request

#### `GET /sync/{{ code }}`

Returns a [SyncRequest][] based on the given code parameter. 

## Usage

### Adding a device to an existing group

#### Start Conditions

##### Initiator Client

The Initiator Client is a client who is already a part of a federation of
clients. The Initiator Client wishes to add the Target Client to its federation.
The Initiator Client has the following properties already defined:

- `id`
- `group`
- `name`
- `peers` (if it has any peers)
- `publicKey`

##### Target Client

The Target Client is a fresh client with no group id or peers. The Target Client
comes to the party prepared with:

- `id`
- `name`
- `publicKey`

#### End Result

![Sync Flow](https://rawgit.com/clippy-io/clippy-acceptance/master/assets/SyncProcessBeforeAfter.svg)

##### Initiator Client

Initiator client adds the Target Client to it's `peers` array. 

##### Target Client

Target Client inherits the Initiator's `group` id, which means it joins the
Initiator Client's federation. The Target Client also inherits all of the
Initiator Client's peers.

#### Flow

![Sync Flow](https://rawgit.com/clippy-io/clippy-acceptance/master/assets/SyncProcess.svg)

## Todo

- :soon: Sending Messages Between Clients

[Client]: #Client
[Error]: #Error
[SemVer]: http://semver.org/
[SyncCode]: #SyncCode
[SyncRequest]: #SyncRequest
[UUIDV4]: http://www.ietf.org/rfc/rfc4122.txt
[node-config]: https://github.com/lorenwest/node-config
[rfc4880]: http://tools.ietf.org/html/rfc4880
