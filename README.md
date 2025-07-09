const { Client, PrivateKey, TokenCreateTransaction, TokenType, 
        TokenMintTransaction, TopicCreateTransaction } = require("@hashgraph/sdk");

// Configuration - YOUR CREDENTIALS
const OPERATOR_ID = "0.0.6255880";
const OPERATOR_KEY = "3030020100300706052b8104000a04220420d7a207928653131acc3068bd64d0c2e6d7ca154a5111d2dbbc61fdb9ce73b52d";

// Setup client
const client = Client.forTestnet();
client.setOperator(OPERATOR_ID, PrivateKey.fromString(OPERATOR_KEY));

// Generate compressed energy data
function generateEnergyData() {
  const time = Date.now().toString().slice(-6);
  const kWh = Math.floor(Math.random() * 1000);
  const flow = 500 + Math.floor(Math.random() * 200);
  const eff = 750 + Math.floor(Math.random() * 150);
  const plant = Math.floor(1000 + Math.random() * 9000);
  const lat = Math.floor(Math.random() * 9000);
  const lon = Math.floor(Math.random() * 18000);
  return `${time}:${kWh}:${flow}:${eff}:${plant}:${lat}:${lon}`;
}

// Create HCS Topic
async function createAuditLog() {
  const topicTx = await new TopicCreateTransaction()
    .setTopicMemo("HydroData")
    .execute(client);
  return (await topicTx.getReceipt(client)).topicId.toString();
}

// Create NFT Token
async function createNFTToken() {
  const tokenTx = await new TokenCreateTransaction()
    .setTokenName("HydroCert")
    .setTokenSymbol("HYDRO")
    .setTokenType(TokenType.NonFungibleUnique)
    .setTreasuryAccountId(OPERATOR_ID)
    .setSupplyKey(PrivateKey.fromString(OPERATOR_KEY).publicKey)
    .execute(client);
  return (await tokenTx.getReceipt(client)).tokenId.toString();
}

// Mint NFT
async function mintCertificate(tokenId, compressedData, topicId) {
  const metadata = JSON.stringify({t: topicId, d: compressedData});
  const mintTx = await new TokenMintTransaction()
    .setTokenId(tokenId)
    .setMetadata([Buffer.from(metadata)])
    .execute(client);
  return (await mintTx.getReceipt(client)).serials[0].toString();
}

// Main deployment
async function deploySystem() {
  console.log("🚀 Starting Hydropower Certificate Deployment");
  
  try {
    // Execute all steps
    const topicId = await createAuditLog();
    const data = generateEnergyData();
    const tokenId = await createNFTToken();
    const serial = await mintCertificate(tokenId, data, topicId);
    
    // Success output
    console.log("✅ DEPLOYMENT SUCCESSFUL!");
    console.log("========================");
    console.log(`📖 Audit Log: https://hashscan.io/testnet/topic/${topicId}`);
    console.log(`🖼️ NFT Certificate: https://hashscan.io/testnet/token/${tokenId}?serial=${serial}`);
    
    // View your account transactions
    console.log("\n🔍 View Account Transactions:");
    console.log(`https://hashscan.io/testnet/account/${OPERATOR_ID}`);
    
    // Force immediate exit to avoid timeout warning
    return process.exit(0);
    
  } catch (error) {
    console.error("❌ Error:", error.message);
    process.exit(1);
  }
}

// Start deployment
deploySystem();
