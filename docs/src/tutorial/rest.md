# Via the REST API

::: danger

Note: This site is work-in-progress and the mainnet library is currently in a prototype stage. 
Things may and will change randomly. There is no backward compatibility guarantee yet, 
even though we try not to break things too often. Use at your own risk. To see the old site with the full plan 
(yet to be implemented), go [here](https://web.archive.org/web/20200810182937/https://mainnet.cash/).

:::

## Let's get programming

Note that this tutorial describes calling the REST API approach (for server-side programming languages, like PHP, Go, Java). 
Alternatively, see [JavaScript (browser)](/tutorial/). 

We generate bindings and packages for some programming languages, so that you don't have
to do the REST calls manually, see [here](/tutorial/other-languages.html). You can generate bindings for nearly 
every other programming language easily.

**You can find the full latest REST API reference at [https://rest-unstable.mainnet.cash/api-docs/](https://rest-unstable.mainnet.cash/api-docs/).**

Let's first create a test wallet:

```bash
curl -X POST https://rest-unstable.mainnet.cash/wallet/create \
    -H "Content-Type: application/json" \
    -d '{"type": "seed", "network": "testnet"}'
```

Response:

```json
{
  "name": "",
  "cashaddr": "bchtest:qrau3n8tzcv2a4yqsr603unhxx4vp9ph0yg2g9449d",
  "walletId": "seed:testnet:table later ... stove kitten pluck:m/44'/0'/0'/0/0",
  "network": "testnet"
}
```

This creates a **TestNet** wallet.  This has the cashaddress of the wallet, where you can send money, and the `walletId`. 
Note the `walletId` - we're going to need it later. This wallet will not be persisted. See below for persistent wallets. 

::: tip rest-unstable

This tutorial shows calls to `https://rest-unstable.mainnet.cash`, which as the name implies is **unstable** by design.
You can use it to learn, but for production you are expected to [run your own service](/tutorial/running-rest.md), because
otherwise you actually send us your _private keys_, which is absolutely insecure.

:::

::: danger walletId contains the private key

Keep `walletId` a secret as it contains the private key that allows spending from this wallet. Seed phrase, WIF - all
these contain the private key. 

:::

::: tip What is TestNet? Where to get TestNet money and a wallet?

`TestNet` is where you test your application. TestNet money has no price. Opposite of TestNet is `MainNet`, 
which is what people usually mean when they talk about Bitcoin Cash network.  

You can get free TestNet money [here](https://faucet.fullstack.cash/).


If you need a wallet that supports the TestNet, download [Electron Cash](https://electroncash.org/) and 
run it using `electron-cash --testnet` flag. For example, on MacOS that would be:

`/Applications/Electron-Cash.app/Contents/MacOS/Electron-Cash --testnet`


:::

To create a MainNet wallet (Bitcoin Cash production network): 

```bash
curl -X POST https://rest-unstable.mainnet.cash/wallet/create \
    -H "Content-Type: application/json" \
    -d '{"type": "seed", "network": "mainnet"}'
```

Response:

```json
{
  "name": "",
  "cashaddr": "bitcoincash:qr70hzy3sfmrcknd7apksmxhhf48cwkuavckm7lt8f",
  "walletId": "seed:mainnet:cave blue ... skill shoot faculty:m/44'/0'/0'/0/0",
  "network": "mainnet"
}
```

Seed phrase wallets use the derivation path `m/44'/0'/0'/0/0` by default (Bitcoin.com wallet compatibility)

If you want to manually construct a walletId from a WIF (a private key), just build a string like this:

```
wif:mainnet:WIFHERE
```

Similarly, from a seed and a derivation path:

```
seed:mainnet:SEED WORDS HERE:m/DERIVATION/PATH
```

Networks: `mainnet`, `testnet`, `regtest` ([see below](#regtest-wallets))

## Named wallets (persistent)

Named wallets are used to save the private key inside the REST API server, so that you 
can refer to it by name and always get the same wallet.

To create a persistent wallet:

```bash
curl -X POST https://rest-unstable.mainnet.cash/wallet/create \
    -H "Content-Type: application/json" \
    -d '{"type": "seed", "network": "testnet", "name": "wallet_1"}'
```

Response:

```json
{
  "name": "wallet_1",
  "cashaddr": "bchtest:qp3wsgxkeezy3rumwzja0yxlmgra2jt74ymtrmayyl",
  "walletId": "named:testnet:wallet_1",
  "seed": {
    "seed": "parade vocal foil key orchard pact mansion arena bounce caught perfect true",
    "derivationPath": "m/44'/1'/0'/0/0"
  },
  "network": "testnet"
}
```

The wallet's private key will be stored in the PostgreSQL database of the REST API server. 

Note that `rest-unstable.mainnet.cash` will not allow you to store mainnet private keys, 
you need to [run your own service](/tutorial/rest.md) for that. We really don't want to store your private keys! 

## Getting the balance

To get the balance of your wallet you can do this (use the `walletId` that you got previously):

```bash
curl -X POST https://rest-unstable.mainnet.cash/wallet/balance \
  -H "Content-Type: application/json" \
  -d '{
    "walletId": "named:testnet:wallet_1"
  }'
```

Response:

```json
{"bch": 0, "sat": 0, "usd": 0}
```

Or you can use `unit` in the call to get just the number:

```bash
curl -X POST https://rest-unstable.mainnet.cash/wallet/balance \
  -H "Content-Type: application/json" \
  -d '{
    "walletId": "named:testnet:wallet_1", 
    "unit": "sat"
  }'
```

Response:

```
0
```

You can ask for `usd`, `sat`, `bch` (or `satoshi`, `satoshis`, `sats` - just in case you forget the exact name).

- 1 satoshi = 0.00000001 Bitcoin Cash (1/100,000,000th)
- 1 Bitcoin Cash = 100,000,000 satoshis

`USD` returns the amount at the current exchange rate. 


### Watch-only wallets

You can find out a balance of any cashaddr (say `bchtest:qq1234567`) by building a `walletId` like this:

```
watch:testnet:bchtest:qq1234567
```

...and then doing the regular `wallet/balance` query.

## Sending money

Let's create another wallet and send some of our money there. 

Remember, that first you need to send some satoshis to the cashaddr of your original wallet (see the TestNet note above).

Check that the balance is there in the original wallet:

```bash
curl -X POST https://rest-unstable.mainnet.cash/wallet/balance \
-H "Content-Type: application/json" \
-d '{"walletId": "wif:testnet:cRqxZECspKgkuBbdCnnWrRsMsYLUeTWULYRRW3VgHKedSMbM6SXB"}'
```

```json
{"bch": 0.000100000, "sat": 10000, "usd": 0.02}
```

Now, we can send 100 satoshis to the `...z2pu` address

```bash
curl -X POST https://rest-unstable.mainnet.cash/wallet/send \
  -H "Content-Type: application/json" \
  -d '{
    "walletId": "wif:testnet:cRqxZECspKgkuBbdCnnWrRsMsYLUeTWULYRRW3VgHKedSMbM6SXB",
    "to": [
      {
        "cashaddr": "bchtest:qz73ul3zy8mvpjt4upw64n58t8gt2ru735qn0dz2pu",
        "value": 100,
        "unit": "sat"
      }
    ]
  }'
```

Note that you can send to many addresses at once.

Response:

```json
{
  "txId": "316f923a1f4c47ac6562779fe6870943eec4f98a622a931f2cc1acd0790ebd69",
  "balance": {"bch": 0.00009680, "sat": 9680, "usd": 0.02}
}
```

You get the transaction ID (txid) that [you can see on the TestNet block explorer](https://explorer.bitcoin.com/tbch/tx/316f923a1f4c47ac6562779fe6870943eec4f98a622a931f2cc1acd0790ebd69)
and the balance left in the original wallet.

Let's print the balance of `...z2pu` wallet:

```bash
{"bch": 0.00000100, "sat": 100, "usd": 0.00}
```

Great! You've just made your first transaction!

Now you can send all of your money somewhere else:

```bash
curl -X POST https://rest-unstable.mainnet.cash/wallet/send_max \
  -H "Content-Type: application/json" \
  -d '{
    "walletId": "wif:testnet:cRqxZECspKgkuBbdCnnWrRsMsYLUeTWULYRRW3VgHKedSMbM6SXB",
    "cashaddr": "bchtest:qz73ul3zy8mvpjt4upw64n58t8gt2ru735qn0dz2pu"
  }'
```

This will send the maximum amount (minus the transaction fees of 1 satoshi per byte, there are usually 200-300 bytes per transaction).

## Waiting for a transaction

### QR codes

Let's say you want to display a QR code for you user to pay you money and show an alert when money arrives?

Let's display a QR code:

```shell script
curl -X POST https://rest-unstable.mainnet.cash/wallet/deposit_qr \
  -H "Content-Type: application/json" \
  -d '{
    "walletId": "wif:testnet:cRqxZECspKgkuBbdCnnWrRsMsYLUeTWULYRRW3VgHKedSMbM6SXB"
  }'
```

Response:

```json
{"src":"data:image/svg+xml;base64,PD94bWwgdmVyc2...."}
```

You can use this `src` directly in the image tag:

```html
<p style="text-align: center;">
    <img src="data:image/svg+xml;base64,PD94bWwgdmVyc2...." style="width: 15em;">
</p>
```

### Waiting

Currently, the only way to wait for a tx is to poll the balance (this will improve later)

## Escrow contracts

::: warning 
Not fully tested yet
:::

Ok, let's now assume that you are building a service where you want to connect a buyer and a seller (a freelance marketplace 
or a non-custodial exchange), but at the same time you don't want to hold anyone's money, 
but only act as an arbiter in case something goes wrong. It's possible in Bitcoin Cash and it's called "an escrow contract".

You'll need three addresses for this: `buyer`, `seller` and `arbiter`. 

Note: The source for the contract is [here](https://github.com/mainnet-cash/mainnet-js/blob/a35c3ff9590eb04e03b122d7bb2d4bbb12150d66/src/contract/escrow/EscrowContract.ts#L98-L118). 

1) Buyer sends the money to the contract and could then release the money to the seller 
2) The seller could refund the buyer
3) Arbiter can either complete the transaction to the seller or refund to the buyer, but cannot steal the money 

```bash
curl -X POST https://rest-unstable.mainnet.cash/contract/escrow/create \
  -H "Content-Type: application/json" \
  -d '{
  "buyerAddr": "bchtest:qrnluuge56ahxsy6pplq43rva7k6s9dknu4p5278ah",
  "arbiterAddr": "bchtest:qzspcywxmm4fqhf9kjrknrc3grsv2vukeqyjqla0nt",
  "sellerAddr": "bchtest:qz00pk9lfs0k9f5vf3j8h66qfmqagk8nc56elq4dv2",
  "amount": 10000,
  "nonce": 808
}'
```

Response:

```json
{
  "contractId": "testnet␝MTU4LDI0MCwyMTYsMTkxLDc2LDMxLDk4LDE2NiwxNDAsNzYsMTAwLDEyMywyMzUsNjQsNzgsMTkzLDIxMiw4OCwyNDMsMTk3␞MjMxLDI1NCwxMTMsMjUsMTY2LDE4NywxMTUsNjQsMTU0LDgsMTI2LDEwLDE5NiwxMDgsMjM5LDE3MywxNjgsMjEsMTgyLDE1OQ==␞MTYwLDI4LDE3LDE5OCwyMjIsMjM0LDE0NCw5MywzNywxODAsMTM1LDEwNSwxNDMsMTcsNjQsMjI0LDE5Nyw1MSwxNTAsMjAw␞MTAwMDA=␞ODA4␝cHJhZ21hIGNhc2hzY3JpcHQgXjAuNS4zOwogICAgICAgICAgICBjb250cmFjdCBlc2Nyb3coYnl0ZXMyMCBzZWxsZXJQa2gsIGJ5dGVzMjAgYnV5ZXJQa2gsIGJ5dGVzMjAgYXJiaXRlclBraCwgaW50IGNvbnRyYWN0QW1vdW50LCBpbnQgY29udHJhY3ROb25jZSkgewoKICAgICAgICAgICAgICAgIGZ1bmN0aW9uIHNwZW5kKHB1YmtleSBzaWduaW5nUGssIHNpZyBzLCBpbnQgYW1vdW50LCBpbnQgbm9uY2UpIHsKICAgICAgICAgICAgICAgICAgICByZXF1aXJlKGhhc2gxNjAoc2lnbmluZ1BrKSA9PSBhcmJpdGVyUGtoIHx8IGhhc2gxNjAoc2lnbmluZ1BrKSA9PSBidXllclBraCk7CiAgICAgICAgICAgICAgICAgICAgcmVxdWlyZShjaGVja1NpZyhzLCBzaWduaW5nUGspKTsKICAgICAgICAgICAgICAgICAgICByZXF1aXJlKGFtb3VudCA+PSBjb250cmFjdEFtb3VudCk7CiAgICAgICAgICAgICAgICAgICAgcmVxdWlyZShub25jZSA9PSBjb250cmFjdE5vbmNlKTsKICAgICAgICAgICAgICAgICAgICBieXRlczM0IG91dHB1dCA9IG5ldyBPdXRwdXRQMlBLSChieXRlczgoYW1vdW50KSwgc2VsbGVyUGtoKTsKICAgICAgICAgICAgICAgICAgICByZXF1aXJlKGhhc2gyNTYob3V0cHV0KSA9PSB0eC5oYXNoT3V0cHV0cyk7CiAgICAgICAgICAgICAgICB9CgogICAgICAgICAgICAgICAgZnVuY3Rpb24gcmVmdW5kKHB1YmtleSBzaWduaW5nUGssIHNpZyBzLCBpbnQgYW1vdW50LCBpbnQgbm9uY2UpIHsKICAgICAgICAgICAgICAgICAgICByZXF1aXJlKGhhc2gxNjAoc2lnbmluZ1BrKSA9PSBhcmJpdGVyUGtofHxoYXNoMTYwKHNpZ25pbmdQaykgPT0gc2VsbGVyUGtoKTsKICAgICAgICAgICAgICAgICAgICByZXF1aXJlKGNoZWNrU2lnKHMsIHNpZ25pbmdQaykpOwogICAgICAgICAgICAgICAgICAgIHJlcXVpcmUoYW1vdW50ID49IGNvbnRyYWN0QW1vdW50KTsKICAgICAgICAgICAgICAgICAgICByZXF1aXJlKG5vbmNlID09IGNvbnRyYWN0Tm9uY2UpOwogICAgICAgICAgICAgICAgICAgIGJ5dGVzMzQgb3V0cHV0ID0gbmV3IE91dHB1dFAyUEtIKGJ5dGVzOChhbW91bnQpLCBidXllclBraCk7CiAgICAgICAgICAgICAgICAgICAgcmVxdWlyZShoYXNoMjU2KG91dHB1dCkgPT0gdHguaGFzaE91dHB1dHMpOwogICAgICAgICAgICAgICAgfQogICAgICAgICAgICB9CiAgICAgICAg",
  "cashaddr": "bchtest:pz77d0jkycc6kfrv5cdvxg5afd8qnlj495tfqt60hs"
}
```

The `contractId` has all the data needed about the contract (but it's pretty big). Store it in your database, so
that you can execute all the necessary functions later.

You can now send money to the contract (just use the `cashaddr` from the response) and check the balance of the contract.

Note: Escrow contract is big (in bytes) and requires a big fee, so the minimum that you can send to it is about 3700 satoshis. 

Now, we can execute the necessary functions:

1) Buyer releases the funds
```bash
curl -X POST https://rest-unstable.mainnet.cash/contract/escrow/call \
  -H "Content-Type: application/json" \
  -d '{
    "contractId": "....",
    "walletId": "seed:testnet:....",
    "action": "spend"
  }'
```

You can optionally provide `"getHexOnly": true` flag if you want to get the raw transaction to broadcast later, otherwise the
transaction will be broadcast immediately. You can also specify the exact UTXO IDs to spend 
(you can use [utxos call](https://rest-unstable.mainnet.cash/api-docs/#/contract/escrowUtxos) to list them):

```
...
"utxoIds": [ 
    "1e6442a0d3548bb4f917721184ac1cb163ddf324e2c09f55c46ff0ba521cb89f:0" 
]
...
```


2) Seller refunds
```bash
curl -X POST https://rest-unstable.mainnet.cash/contract/escrow/call \
  -H "Content-Type: application/json" \
  -d '{
    "contractId": "....",
    "walletId": "seed:testnet:....",
    "action": "refund"
  }'
```

3) Arbiter releases the funds or refunds 

(the same: spend or refund, just replace the walletId with the arbiter's one)

## Utilities

Certain tools common in bitcoin-like currencies may not be in a standard library.

### Currency conversions

Need to find out how many BCH are there currently in 1 USD, or find out how many satoshis are there in 100 USD? Easy!

```bash
curl -X POST https://rest-unstable.mainnet.cash/util/convert \
  -H "Content-Type: application/json" \
  -d '{
  "value": 100,
  "from": "usd",
  "to": "sat"
}'

```
returns something like:

```
...
28067024
...
```

## CashScript

Somewhat done, but not yet documented...

## RegTest wallets

During the local development and testing, you might not want to use TestNet coins, so you can use so-called "RegTest wallets".

::: tip What is RegTest?

`RegTest` is a mode, in which you can run your Bitcoin Cash node locally, and you can get as many test coins as you need,
but they exist on your machine only. RegTest wallets are supported by the mainnet library.

:::

A full Bitcoin node, an Electrum server and open Postgres server configuration is available for testing in a
Docker Compose file at `jest/regtest-docker-compose.yml`

It can be brought up with:

```bash
./jest/docker/start.sh 
```

To stop it:

```bash
./jest/docker/stop.sh
```

The Electrum server (Fulcrum) is available at `ws://127.0.0.1:60003` on your local machine.  
The regtest BCHN node is on port `18443` available with RPC using credentials in `.env.regtest`.
An open Postgres server is also available on port `15432`

To use this wallet from your code, just use `network`: `RegTest` in your calls.

## Webhooks

Webhooks are custom callbacks which notify user upon certain actions. Mainnet provides the webhook functionality to monitor addresses and transactions.

You can register a webhook with the following `curl` call:

```bash
curl -X POST "https://rest-unstable.mainnet.cash/webhook/watch_address" \
  -H  "accept: application/json"
  -H  "Content-Type: application/json"
  -d '{
    "cashaddr":"bchtest:qzd0tv75gx6y0zspzwqpgkwkq0n72g8fsq2zch26s2",
    "url":"http://example.com/webhook",
    "type":"transaction:in,out",
    "recurrence":"recurrent",
    "duration_sec":86400
  }'
```

The endpoint you specify in the `url` parameter will be called with HTTP POST method and data with the webhook response. After the `duration_sec` seconds the registered webhook will expire and you stop receiving notifications.

Webhooks can be set up to fire once and expire if the `recurrence` parameter is set to `once` or continuously until they expire by the timeout.

See full webhook rest documentation in the [Swagger UI](https://rest-unstable.mainnet.cash/api-docs/#/webhook/watchAddress)

## WebSocket API reference

We provide some functionality over websockets where traditional REST servers would timeout. Examples are waiting for transactions and watching balances.
Websockets allow to subscribe to server events, sending responses and notifications asynchronously.

Check out the jsfiddle [demo](https://jsfiddle.net/Lm87yzuj/6/)

The client-server communication is in JSON format.

Server expects the messages in the following form:
```json
{
  method: "methodName",
  data: {
    dataMember: "dataValue"
  }
}
```

Responses are JSON objects, depending on the method invoked. If an error is encountered the response will contain the error message:
```json
{
  "error": "Mainnet websockets: unsupported method watchBirds"
}
```

### Supported methods
#### watchBalance

Receive notification upon the address' balance change. Responds recurrently.
Request:
```json
{
  method: "watchBalance",
  data: {
    cashaddr: "bitcoincash:qzxzl07tth5qx4shphrpzz38wnstwac5ksqnc6yyr3"
  }
}
```

Response:
```json
{
  bch: 0.00009383,
  sat: 9383,
  usd: 0.029358468699999998
}
```

#### waitForTransaction

Waits for the next transaction of the address. Responds once.
```json
{
  method: "waitForTransaction",
  data: {
    cashaddr: "bitcoincash:qzxzl07tth5qx4shphrpzz38wnstwac5ksqnc6yyr3"
  }
}
```

Response: Raw transaction in verbose format as per [specification](https://electrum-cash-protocol.readthedocs.io/en/latest/protocol-methods.html#blockchain-transaction-get)
```json
{
  hash: '0f617203d936386c4ed6a828f911e577394cd753b652745d5207fbb139c0d924',
  hex: '02000000011f6f174a63236d4d32e256dbf92b86ee5fc37fdd11b9ce82ad1950bf2e799f2100000000644124778cbd7e22a2e4ddbc06901205f3c818a380945c696d012aae47dee8dd1fdfe7edfda93d165ceaa53501690d268d1acae405c7e635e864be71bbc8e6264f1a412103410ef048b3da351793f6ed14cc2fde460becc5b658d9138443b9a3000707a6a70000000002e8030000000000001976a91446df42ac56b164b81f36a93210a04a30293659aa88ac3ced052a010000001976a91456b6b22042b90dd67bf2fbfb9aff7d37fbee112488ac00000000',
  locktime: 0,
  size: 219,
  txid: '0f617203d936386c4ed6a828f911e577394cd753b652745d5207fbb139c0d924',
  version: 2,
  vin: [
    {
      scriptSig: [Object],
      sequence: 0,
      txid: '219f792ebf5019ad82ceb911dd7fc35fee862bf9db56e2324d6d23634a176f1f',
      vout: 0
    }
  ],
  vout: [
    { n: 0, scriptPubKey: [Object], value: 0.00001 },
    { n: 1, scriptPubKey: [Object], value: 49.9999878 }
  ]
}
```