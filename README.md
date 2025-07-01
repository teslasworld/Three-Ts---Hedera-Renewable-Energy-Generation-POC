# Hedera-POC 

````markdown
# Real‑World Deployment Guide: TTT Hydro‑Powered PoC on Hedera

This is the end‑to‑end production‑ready deployment for the TTT (True Tesla Technologies) hydro‑powered IoT proof of concept. It covers:

1. Simulated sensor data  
2. IPFS storage via Web3.Storage  
3. Hedera Consensus Service for immutable logging  
4. Token minting on Hedera (NFT as “energy receipt”)  
5. Production best practices  

---

## Prerequisites

- Hedera Testnet account (via [Hedera Portal](https://portal.hedera.com/register))  
- Node.js v16+  
- Git  
- Code editor (VS Code, etc.)  
- Web3.Storage account  
- Hashscan testnet access  

---

## Setup & Installation

Clone this repository or open it in GitHub.dev:

```bash
git clone https://github.com/teslasworld/Hedera-POC.git
cd Hedera-POC
npm init -y
npm install dotenv @hashgraph/sdk web3.storage buffer
````

---

## 1. Configure Environment

Create a file named `.env` in the project root:

```ini
OPERATOR_ID=0.0.YOUR_HEDERA_ACCOUNT
OPERATOR_KEY=302e0201...    # your DER- or HEX-encoded private key
IPFS_API_TOKEN=YOUR_WEB3_STORAGE_TOKEN
```

---

## 2. Simulate Sensor Data (`sensorSimulator.js`)

```js
function simulateReading() {
  return JSON.stringify({
    deviceId: "hydro-sensor-001",
    timestamp: new Date().toISOString(),
    kWh: (Math.random() * 10).toFixed(2),
    pH: (7 + Math.random() * 0.5).toFixed(1),
    temperature: (20 + Math.random() * 5).toFixed(1)
  });
}

console.log(simulateReading());
```

Run:

```bash
node sensorSimulator.js
```

---

## 3. Upload to IPFS (`ipfsUploader.js`)

```js
require('dotenv').config();
const { Web3Storage, File } = require('web3.storage');
const { Blob } = require('buffer');

async function main() {
  const client = new Web3Storage({ token: process.env.IPFS_API_TOKEN });
  const data = process.argv[2];
  const blob = new Blob([data], { type: 'application/json' });
  const file = new File([blob], 'sensor.json');
  const cid = await client.put([file]);
  console.log(cid);
}

main().catch(console.error);
```

Run:

```bash
node ipfsUploader.js '<sensor JSON>'
```

Verify in you browser:

```
https://dweb.link/ipfs/<CID>
```

---

## 4. Publish to Hedera Topic (`publishReading.js`)

```js
require('dotenv').config();
const {
  Client,
  ConsensusCreateTopicTransaction,
  ConsensusMessageSubmitTransaction
} = require('@hashgraph/sdk');

async function main() {
  const client = Client.forTestnet();
  client.setOperator(process.env.OPERATOR_ID, process.env.OPERATOR_KEY);

  // Create a new topic
  const topicTx = await new ConsensusCreateTopicTransaction().execute(client);
  const topicReceipt = await topicTx.getReceipt(client);
  const topicId = topicReceipt.topicId.toString();

  // Publish a message
  const submitTx = await new ConsensusMessageSubmitTransaction()
    .setTopicId(topicId)
    .setMessage(process.argv[2])
    .execute(client);
  await submitTx.getReceipt(client);

  console.log("Topic ID:", topicId);
  console.log("View on Hashscan:", `https://hashscan.io/testnet/topic/${topicId}`);
}

main().catch(console.error);
```

Run:

```bash
node publishReading.js '<sensor JSON>'
```

---

## 5. Mint NFT (`mintNFT.js`)

```js
require('dotenv').config();
const {
  Client,
  PrivateKey,
  TokenCreateTransaction,
  TokenMintTransaction,
  TokenType,
  AccountId
} = require('@hashgraph/sdk');

async function main() {
  const client = Client.forTestnet();
  const operatorKey = PrivateKey.fromString(process.env.OPERATOR_KEY);
  client.setOperator(AccountId.fromString(process.env.OPERATOR_ID), operatorKey);

  // Create NFT token
  const createTx = new TokenCreateTransaction()
    .setTokenName("TTT Energy Receipt")
    .setTokenSymbol("TTT-NFT")
    .setTokenType(TokenType.NonFungibleUnique)
    .setTreasuryAccountId(AccountId.fromString(process.env.OPERATOR_ID))
    .setSupplyKey(operatorKey.publicKey)
    .freezeWith(client)
    .sign(operatorKey);

  const createResp = await createTx.execute(client);
  const tokenId = (await createResp.getReceipt(client)).tokenId;

  // Mint an NFT with metadata
  const metadata = JSON.stringify({
    cid: process.argv[2],
    data: JSON.parse(process.argv[3])
  });
  const mintTx = new TokenMintTransaction()
    .setTokenId(tokenId)
    .setMetadata([Buffer.from(metadata)])
    .freezeWith(client)
    .sign(operatorKey);

  const mintResp = await mintTx.execute(client);
  const serial = (await mintResp.getReceipt(client)).serials[0].toString();

  console.log("Token ID:", tokenId);
  console.log("Serial:", serial);
  console.log("View on Hashscan:", `https://hashscan.io/testnet/token/${tokenId}?serial=${serial}`);
}

main().catch(console.error);
```

Run:

```bash
node mintNFT.js <CID> '<sensor JSON>'
```

---

## Best Practices (Production)

* Store secrets in a vault or HSM; do not commit `.env`
* Set `.setMaxTransactionFee()` on every transaction
* Batch multiple readings into fewer transactions to save fees
* Implement retry logic with exponential backoff for transient failures
* Monitor Mirror Node confirmations via its REST API
* Sanitize and validate all input data
* Log transaction IDs, fees, receipts, and HBAR balances
* Integrate alerts and dashboards (Grafana, Prometheus, Sentry)

---

## Deployment Checklist

* [ ] Run end-to-end test on Testnet
* [ ] Secure keys and IPFS tokens
* [ ] Enforce max fees and circuit breakers
* [ ] Monitor account balance and topic growth

---

## Useful Links

* Hedera Portal: [https://portal.hedera.com/](https://portal.hedera.com/)
* Web3.Storage: [https://web3.storage/](https://web3.storage/)
* Testnet Explorer: [https://hashscan.io/testnet](https://hashscan.io/testnet)
* IPFS Gateway: [https://dweb.link/ipfs/](https://dweb.link/ipfs/)
* Fee Estimation Tool: [https://github.com/hashgraph/hedera-fee-tool-js](https://github.com/hashgraph/hedera-fee-tool-js)


