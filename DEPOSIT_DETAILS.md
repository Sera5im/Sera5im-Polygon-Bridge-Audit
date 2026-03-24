# 📥 Deposit Flow: Technical Breakdown (L1 → L2)

This section provides a deep-dive analysis of the **Lock-and-Mint** mechanism. We trace the asset's path from the user's transaction on Ethereum to the state synchronization event.

---

## 🖼 Deposit Process Visualization

<img width="512" height="272" alt="image" src="https://github.com/user-attachments/assets/838ba0b8-3d68-4963-8da9-05f596dfea85" />

---

## 🔍 Line-by-Line Code Analysis

### 1. RootChainManager.sol — `depositFor`
This is the primary external gateway. It acts as a safety filter before passing the data to the internal engine.

```solidity

function depositFor(
    address user,              // L2 Recipient: Who will receive the assets on Polygon?
    address rootToken,         // Token Address: Which specific asset is being bridged?
    bytes calldata depositData  // Flexible Payload: Amount (ERC20) or Token ID (ERC721)
) external override {
    // [Check] Native Ether Filter
    // ETH is handled via a different flow; it is not a standard ERC20.
    require(
        rootToken != ETHER_ADDRESS,
        "RootChainManager: INVALID_ROOT_TOKEN"
    );

    // [Action] Data Forwarding
    // All validated parameters are passed to the internal processing function.
    _depositFor(user, rootToken, depositData);
}
