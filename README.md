# USDX Wallet Exchange API Specification

USDX Wallet Exchange API allows exchanges to interact with USDX Wallet backend for the following purposes:

1. Transfer USDX/LHT assets from exchange account to user account.
2. Receive notifications when user transfers assets from his account to exchange account.
3. Get transactions history for exchange account.
4. Get exchange account balance.

## Exchange Registration

To register your exchange, please send request to api@usdx.cash, providing information specified below:

1. Exchange name.
2. Callback URL. USDX Wallet backend will send notifications about transfer transactions to this URL ([see Callback endpoint description](#callback-endpoint)).
3. Memo regexp (see details in the [Identifiying user transaction](#identifiying-user-transaction)).
4. Exchange logo (PNG, min 256x256), will be displayed in our mobile wallet.
5. (Optional). The list of exchange IP addresses. If specified, USDX Wallet backend will accept requests coming only from these whitelisted IP addresses.

After registration you will be able to receive USDX/LHT assets on the provided account from users of USDX Wallet as well as send assets from this account to users via USDX Wallet Exchange API.

As the result of the registration, you will receive the following information:

1. Exchange ID. Unique identifier of your exchange used by USDX Wallet backend. Should be kept in secret.
2. API key. Unique key for your exchange used to sign requests. Should be kept in secret.
3. Account name. Name of your exchange account in USDX Wallet blockchain. Let your exchange users know this account name, so they will be able to send USDX/LHT assets to it (via USDX Wallet mobile application).

## General API information

API is implemented as HTTP REST endpoints.
Endpoints are described in [API Endpoints Reference](#api-endpoints-reference) section.
URLs of all endpoints start with `${BASE_URL}`, which should be requested from USDX Wallet team.

POST requests should contain body in JSON format, UTF-8 encoded.

All HTTP requests must be signed as specified in [Authentication section](#authentication).

## Authentication

All API HTTP requests must contain `x-usdx-signature` header with request signature.
The signature is generated using the following steps:
1. Take `body` of your prepared HTTP request. Body should be UTF-8 encoded. In case of empty body or `GET`-request, `body` is considered as empty string.
2. Get current `timestamp` (in milliseconds).
3. Take `apiKey`, API key received during registration, [see Exchange Registration](#exchange-registration).
4. Concatenate `body` + `apiKey` + `timestamp`.
5. Calculate SHA-256 hash of the resulted string and take hex representation of it.
6. Set the value of `x-usdx-signature` header in the following format:
```
x-usdx-signature: t=1549022587251, v1=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd
```
where `t` value is `timestamp` used in the calculation, `v1` value is SHA-256 hash (`v` stands for current signature schema, actual version is 1).

Please pay attention that each next API call should have `timestamp` value larger that the value used in the previous call.
In case if it is smaller or equal to `timestamp` value used in the previous call, request will be rejected.

#### Example of request signature calculation:

1. For example, `body` of HTTP request is the following:
```json
{
    "transferId" : "abcdef0123456789abcdef0123456789",
    "accountName": "someAccountName",
    "amount": 1000.23,
    "currency": "USDX",
    "type": "OUTGOING",
    "createdAt": "2019-01-02T08:02:13",
    "status": "PENDING",
    "memo": "memo text",
    "customData": "1cad7e77a0e56ff536d0c",
}
```
2. Current `timestamp` (in milliseconds) is `1546416133123`.
3. `apiKey` is `a1b2c3d4e5f6g7h8`.
4. Result of `body` + `apiKey` + `timestamp` concatenation:
```json
{
    "transferId" : "abcdef0123456789abcdef0123456789",
    "accountName": "someAccountName",
    "amount": 1000.23,
    "currency": "USDX",
    "type": "OUTGOING",
    "createdAt": "2019-01-02T08:02:13",
    "status": "PENDING",
    "memo": "memo text",
    "customData": "1cad7e77a0e56ff536d0c",
}1546416133123a1b2c3d4e5f6g7h8
```
5. Result of SHA-256 hash (hex representation):
```
9ee36fa6b574f6a6afb6525aa9857d5b083ccb5a5c0cfbc1341c135ee764956a
```
6. Value of `x-usdx-signature` header:
```
x-usdx-signature: t=1546416133123, v1=9ee36fa6b574f6a6afb6525aa9857d5b083ccb5a5c0cfbc1341c135ee764956a
```

## API Endpoints Reference

### Response structure

Response structure of all endpoints corresponds to JSend format ([see specification](https://github.com/omniti-labs/jsend)).

In case of success, the structure of response is the following:
```json
{
    "data": {...},
    "message": null,
    "status": "success"
}
```
- `data` contains JSON object specific for this request;
- `message` is `null`;
- `status` is `success`;

In case of errors caught and handled by server (for example, signature is incorrect, there are not enough assets to transfer etc.), the structure of response is the following:
```json
{
    "data": {
        "errorCode": "INCORRECT_SIGNATURE",
        "failReason": "Signature is incorrect"
    },
    "message": null,
    "status": "fail"
}
```
- `errorCode` contains code of the error;
- `failReason` contains error description;
- `message` is `null`;
- `status` is `fail`;

In case of critical errors of the backend itself (for example, internal server error), the structure of response is the following:
```json
{
    "message": "Internal server error",
    "status": "error",
}
```
- `message` contains error description;
- `status` is `error`;

#### List of error codes

The following is the list of possible `errorCode` values in case when response `status` value is `fail`.

|Code                          |Description
|:-----------------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| SIGNATURE_NOT_SPECIFIED      | `x-usdx-signature` request header is absent                                                                                                                                                |
| SIGNATURE_FORMAT_INVALID     | `x-usdx-signature` request header is specified, but has incorrect format                                                                                                                   |
| TIMESTAMP_INVALID            | Timestamp specified in `x-usdx-signature` request header is invalid (previous API call contained larger value of timestamp)                                                                |
| SIGNATURE_INVALID            | Signature specified in `x-usdx-signature` request header is invalid                                                                                                                        |
| IP_ADDRESS_INVALID           | IP address the request is coming from is not whitelisted for this exchange (only in case if exchange provided IP addresses white list, [see Exchange Registration](exchange-registration)) |
| EXCHANGE_ID_INVALID          | Specified `exchangeId` parameter doesn't correspond to any registered exchange                                                                                                             |
| USER_ACCOUNT_INVALID         | User account with specified name doesn't exist                                                                                                                                             |
| ASSETS_NOT_ENOUGH            | Attempt to transfer more assets than exchange account has                                                                                                                                  |
| TRANSACTION_NOT_FOUND        | Transaction with specified ID was not found                                                                                                                                                |

### Object reference

#### Transfer transaction object

Transfer transaction JSON object is used in different API endpoints.
It contains all information about the transfer:

```json
{
    "transferId" : "jdk5823kdywi57kar35dkryqr354mvlf",
    "accountName": "someAccountName",
    "amount": 1000.23,
    "currency": "USDX",
    "type": "OUTGOING",
    "createdAt": "2019-01-02T08:02:13",
    "status": "PENDING",
    "memo": "memo text",
    "customData": "1cad7e77a0e56ff536d0c",
}
```

- `transferId` - ID of the transfer transaction, which can be used, for example, to track transaction status;
- `accountName` - name of account; if `type` is `OUTGOING`, this is the name of user account, which receives assets from the exchange; if `type` is `INCOMING`, this is the name of user account, which transfers assets to the exchange account;
- `amount` - amount of assets transferred by this transaction;
- `currency` - name of assets transferred by this transaction (for the list of possible values, [see Asset name](#asset-name))
- `type` - type of the transfer transaction (for the list of possible values, see [see Transaction type](#transaction-type))
- `createdAt` - date/time in UTC timezone when transaction was created; format is ISO-8601 `YYYY-MM-DDTHH:mm:SS`;
- `status` - current status of the transaction (for the list of possible values, see [see Transaction status](#transaction-status))
- `memo` (optional) - arbitrary text, description of transfer;
- `customData` (optional) - can be present in case if `type` is `OUTGOING` only; arbitrary string with any additional information provided by the exchange;

#### Asset name

There are only two possible values of asset name (`currency` field):
1. LHT
2. USDX

Any another value is considered as incorrect one.

#### Transaction type

Possible types of transaction are:
1. INCOMING - transfer transaction from user account to exchange account.
2. OUTGOING - transfer transaction from exchange account to user account.

#### Transaction status

Possible values of transaction status are:
1. PENDING - transaction was created, but not executed yet.
2. SUCCESS - transaction was successfully executed, assets were transferred to the account.
3. FAIL - transaction failed for some reason (for example, there are not enough assets on the sender's account).

### 1. Transfer assets to user account

Transfers USDX/LHT assets from exchange account to user account.

**URL:** `${BASE_URL}/v1/exchange/:exchangeId/transfer`
- `:exchangeId` - ID of your exchange received as a result of exchange registration [see Exchange Registration](#exchange-registration)

**Method:** `POST`

**Headers:**
`Content-Type: application/json`

`x-usdx-signature: ...` ([see Authentication](#authentication))

**Request body:**
```json
{
    "accountName": "user-account-name",
    "amount": 10000.23,
    "currency": "USDX",
    "memo": "memo text",
    "customData": "1cad7e77a0e56ff536d0c",
}
```

- `accountName` - name of user account in USDX Wallet blockchain, who will receive assets;
- `amount` - amount of assets to be transferred;
- `currency` - name of assets to be transferred  [see Asset name](#asset-name) for the list of possible values;
- `memo` (optional) - arbitrary text, description of transfer transaction, which receiver could see in transaction details in USDX Wallet mobile application;
- `customData` (optional) - arbitrary string with any additional information, that you can attach to the transaction;

**Success response body:**
```json
{
    "data": (Transfer transaction object),
    "message": null,
    "status": "success"
}
```

In case of success, `data` field contains transfer transaction object ([see description](#transfer-transaction-object)) with `status` = `PENDING`.

### 2. Transactions history

Endpoint to retrieve transactions history.
Transactions are retrieved starting from the specified transaction ID (exclusive) towards previous (older) transactions.

**URL:** `${BASE_URL}/v1/exchange/:exchangeId/history?fromId=jdk5823kdywi57kar35dkryqr354mvlf&limit=10`
- `:exchangeId` - ID of your exchange received as a result of exchange registration [see Exchange Registration](#exchange-registration)
- `fromId` (optional) - ID of the transaction, transaction history should be retrieved from (exclusive); if not specified, transactions will be retrieved starting from the most recent one
- `limit` - maximum amount of transactions in the response (maximum possible value is 100)

**Method:** `GET`

**Headers:**
`x-usdx-signature: ...` ([see Authentication](#authentication))

**Success response body:**
```json
{
    "data": {
        "history": [
            (Transfer transaction object),
            (Transfer transaction object),
            ...
        ]
    },
    "message": null,
    "status": "success"
}
```

In case of success, `history` field contains array of transfer transaction objects ([see description](#transfer-transaction-object)), ordered by `createdAt` field descending.
To get next transactions (older ones) use this endpoint again passing `transferId` value of the last transaction as `fromId` parameter.

### 3. Exchange account balance

Endpoint to get current assets balance of exchange account.

**URL:** `${BASE_URL}/v1/exchange/:exchangeId/balance`
- `:exchangeId` - ID of your exchange received as a result of exchange registration [see Exchange Registration](#exchange-registration)

**Method:** `GET`

**Headers:**
`x-usdx-signature: ...` ([see Authentication](#authentication))

**Success response body:**
```json
{
    "status": "success",
    "data": {
        "USDX":1000.23,
        "LHT": 1234.56789
    },
    "message": null
}
```

In case of success, `USDX` and `LHT` fields contains current balance of `USDX` and `LHT` assets on exchange account.

### 4. Transaction status

Endpoint to get current transaction status by its ID.

**URL:** `${BASE_URL}/v1/exchange/:exchangeId/transfer/:transferId`
- `:exchangeId` - ID of your exchange received as a result of exchange registration [see Exchange Registration](#exchange-registration)
- `:transferId` - ID of the transfer transaction the status should be retrieved for

**Method:** `GET`

**Headers:**
`x-usdx-signature: ...` ([see Authentication](#authentication))

**Success response body:**
```json
{
    "data": (Transfer transaction object),
    "message": null,
    "status": "success"
}
```

In case of success, `data` field contains transfer transaction object ([see description](#transfer-transaction-object)) with current status.

### Callback endpoint

You must implement this endpoint on the exchange backend in order to receive notifications from USDX Wallet API backend in the following cases:

1. User executes successful transfer from user's account to exchange account.
2. Status of transaction created by exchange, is changed (from `PENDING` to `SUCCESS` or `FAIL`).

This endpoint must respond with HTTP status 200, no response body is expected.
If endpoint doesn't respond at all or response status code is not 200, USDX Wallet API backend will try to call it again after some time.

It is not guaranteed that even in case when endpoint responds with HTTP status 200, it will be called only once for one transaction. It is possible that duplicated calls will be executed, so endpoint should implement logic to handle such situations properly.

Request to this endpoint contains `x-usdx-signature`, created by USDX Wallet API backend according to the same algorithm as used for USDX Wallet API requests ([see Authentication](#authentication)).
You can check this signature to ensure that the request was really created by USDX Wallet API backend.
This check is optional, however it is recommended for security reasons.

**URL:** can be any; it will be requested by USDX Wallet team during exchange registration process [see Exchange Registration](#exchange-registration).

**Method:** `POST`

**Headers:**
`x-usdx-signature: ...` ([see Authentication](#authentication))

**Request body:**
```
(Transfer transaction object)
```

Request body contains transfer transaction object with current transaction status.

## Identifiying user transaction
When user wants to transfer his tokens from the wallet to the exchange account, he could just make a transaction from his account to the exchange wallet account.
The trickier part is to match transfer from the wallet account 'alice1231' with the exchange account 'alice@example.com'.
For networks like BitCoin and Ethereum exchanges generate uniq wallet address for each incomming transaction.
In the Graphene/Bitshares network each wallet has only one address, so it's not possible to generate 'alias', 'temporary', 'one-off', etc. addresses.
Nevertheless there is a way to match transaction with the user account.

### Memo-based match (using QR code)
Each time user wants to transfer funds to his excange account provide him with the QR code that contains following URL:
```
usdx:exchange-address?amount=100&currency=LHT&memo=11223344&transactionType=strict
```

Where
* `exchange-address` (required) -- transaction destination address, should be your exchange wallet address.
* `currency` (required) -- could be `"LHT"` or `"USDX"`.
* `amount` (optional) -- if specified provides exact amount to be send.
* `memo` -- arbitrary string (up to 100 bytes) that will be passed along with the transaction.
You will receive it in the callback, also you could get it from the transaction history (refer to the API reference).
Memo should contain transferId, account id hash, or any other token that you could use to identify user account later.
* `transactionType` -- type of the transaction, it could be:
  * `"stritct"` -- in this case user can't change transaction details (destination, amount, memo);
  * `"open_amount"` -- user can change amount, but can't change memo or destination;
  * `"normal"` (default) -- no retstrictions, user can change all fields.

When user will scan provided QR code with USDX Wallet app he will be presented with "Send" screen with all the provided data filled in,
some fields could be set as read-only, acording to the `transactionType` value.

For example, if URL
```
usdx:example-exchange?amount=100&currency=LHT&memo=11223344&transactionType=strict
```
is provided, then user will see a "Send" screen
with prefilled fields for destination account, memo, amount and currency, and will only have an option to submit transaction or completely decline it.

If he will be provided with the URL
```
usdx:example-exchange?currency=LHT&memo=11223344&transactionType=open_amount
```
then destination account and memo will be prefilled,
but user could enter arbitrary amount of tokens he would like to send.

If `transactionType` is set to `"normal"` or not set at all, like in
```
usdx:example-exchange?currency=LHT&memo=11223344
```
then fields will be prefilled with specified values, and user will be able to change any of them, like in normal transaction.
You probably won't use this type.

### Memo-based match (using deep link)
Some users use exchange UI on the same (mobile) device where USDX Wallet app is installed,
this prevents them from using QR code.
For that case we have a deep link mechanism that could be used to trigger the app with necessary parameters.
The link looks like
```
https://link.usdx.cash/example-exchange?amount=100&currency=LHT&memo=11223344&transactionType=open_amount
```
Parameters are the same as for QR code.
Please, note, that you CAN'T use provided link as is, each exchange will be provided with uniq URL.

### User experience notes
To provide a good user experience several things should be considered:
* Always provide user with both QR code and deep link.
* Prefer `transactionType=open_amount`, so user would be able to choose arbitrary amount, but will not be able to mess up with memo and account name.
* Provide us with the regexp that we could use to validate the memo, in case if user will decide to fill values manually.
This way our mobile wallet will show a warning to the user, if he will try to send a transaction with wrong code in memo.
For example it could be `/^[0-9]{7}$/` (seven arbitrary digits), or `/^ID[a-z0-9]{4}$/` ('ID' plus four lowercase alphanumeric characters).
Memo couldn't be less than 4 characters.
* API would reject transactions between exchange accounts, so it's a good idea to warn the user if he's trying to withdraw to the
account matching `/^usdx-exchange-.*$/` pattern.
You could find corresponding texts and motivation in our [FAQ](https://usdxwallet.zendesk.com/hc/en-us/articles/360010477580-Why-can-t-I-send-transactions-from-one-exchange-to-another-).
