# Virtual Ledger Clearnet

Clearnet is a mesh network composed of nodes we often refer as message brokers.
They form a layer-3 on top of multiple EVM and non-EVM chains for virtual ledgers.

The Virtual Ledger broker is an off-chain channel provision microservice.

It's a WebSocket web service, where participants first establish an on-chain state-channel, then communicate with the broker
via this channelId (identified by a keccak256). The service can create virtual channels between participants, creating new topics
also identified by a vChanId (keccak256). Associated with each channel is a ledger account with token balance allocation.

The Broker service keeps track of off-chain accounting and persists it on-chain at the request of the participants.

## Authentication and WebSocket Communication

The broker provides a unified WebSocket endpoint for both authentication and communication. The flow works as follows:

1. **Connection**: Client connects to the `/ws` endpoint
2. **Authentication**: The first message must be an authentication message with the following format:

   ```json
   {
     "type": "auth",
     "pub_key": "0x04...", // Public key in hex format
     "message": "Authentication message to sign",
     "signature": "0x..." // Signature of the message using private key
   }
   ```

3. **Regular Communication**: After successful authentication, the client can send various message types:

   ```json
   {
     "type": "subscribe|publish|ping|rpc",
     "channel": "channel-name", // For subscribe/publish
     "data": { ... } // Message payload
   }
   ```

## Software Stack

`go doc -all PKG # To learn about the stack`

- Go lang
- Centrifuge (<https://github.com/centrifugal/centrifuge>)
- Redis CLI v9 (github.com/redis/go-redis/v9)
- SQLite (github.com/mattn/go-sqlite3)
- Gorm (gorm.io/gorm and gorm.io/driver/sqlite)

## Communication flow

Participants can connect with the `broker` identified with their Ethereum public address (40 hex characters)

They can communicate with the broker for instructions related to their on-chain State Channel (using hachi).
A participant can request the broker to create a topic to establish a communication with another participant.

Broker will create a topic room identified with a virtual channel Id (keccak256), participant join the topic
and they will be able to communicate to each other json-rpc messages according to `RPC.md` specification.

By Convention, on participant will be identified as the `Host` whom initiated the creation of the room,
and the other will be the `Guest`.

In the RPC protocol the `Guest` initiate RPC Requests and signs them, while the `Host` will Response to them.
The timestamp in millisecond we will be using is provided by the `Broker`.

## Ledger operations

The broker keep in SQLite database a ledger to keep track of off-chain balance of participant.

### Schema

```sqlite
CREATE TABLE ledger (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    channel_id CHAR(64) NOT NULL,
    participant CHAR(40) NOT NULL,
    token_address CHAR(40) NOT NULL,
    credit INTEGER NOT NULL,
    debit INTEGER NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### Operations

- When Direct Channel opening, we record the participants allocation, creating the initial balance (This information come from blockchain)
- When a Virtual channel is created, some funds are transferred from direct-channel account into virtual-channel account under respective participant
- When Virtual Channel conclude, funds between participant may have move and Broker may have to materialize this outcome on-chain

### Example

1. `Alice` open a channel with broker with 20M SHIB (we credit +100M into her account in the ledger)
2. `Bob` accept and activate the channel with 0 token from his side.
3. `Charlie` open a channel with broker with 100M PEPE
4. `Alice` and `Charlie` establish a Virtual Channel, and agree to allocate respectively 5M SHIB and 10M PEPE (Broker record into the virtual account the transfers between accounts)
5. `Alice` and `Charlie` play a game in their room, `Charlie` win all the pot. the outcome is Alice=0; Charlie=+5M SHIB
6. `Charlie` request settlement to `Broker`; `broker` will re-open a new channel with the settlement. when close ledger balance will be debited

#### Ledger Records

| id | channel_id | participant | token_address | credit | debit | created_at |
|----|------------|-------------|---------------|--------|-------|------------|
| 1 | 0x1234...abc (Alice-Broker) | 0xAlice... | 0xSHIB... | 20000000 | 0 | 2025-03-28 10:00:00 |
| 2 | 0x1234...abc (Alice-Broker) | 0xBob... | 0xSHIB... | 0 | 0 | 2025-03-28 10:05:00 |
| 3 | 0x5678...def (Charlie-Broker) | 0xCharlie... | 0xPEPE... | 100000000 | 0 | 2025-03-28 10:10:00 |
| 4 | 0xabcd...123 (Alice-Charlie vChan) | 0xAlice... | 0xSHIB... | 5000000 | 5000000 | 2025-03-28 10:15:00 |
| 5 | 0xabcd...123 (Alice-Charlie vChan) | 0xCharlie... | 0xPEPE... | 10000000 | 0 | 2025-03-28 10:15:00 |
| 6 | 0xabcd...123 (Alice-Charlie vChan) | 0xCharlie... | 0xSHIB... | 5000000 | 0 | 2025-03-28 10:30:00 |

### Ledger Implementation

An account is uniquely identified by (channel_id, participant, token_address)

#### ledger.go

- File should contain gorm ledger `Entry` model
- Should contain a type `Account` that represent an Account
- Should contain `Ledger` struct which contains gorm.DB with a constructor
- `Ledger` should contain a factory to construct Account objects
- `Account` should have 2 methods, ac.Record To append Entries, and ac.Transfer to move funds to another account
- ac.Record(amount int64) should record a credit if amount > 0, if negative record a debit
