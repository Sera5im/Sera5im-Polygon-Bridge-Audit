# 📥 Deposit Flow: Technical Breakdown (L1 → L2)

This section provides a deep-dive analysis of the **Lock-and-Mint** mechanism. We trace the asset's path from the user's transaction on Ethereum to the state synchronization event.

---

## 🖼 Deposit Process Visualization

<img width="512" height="272" alt="image" src="https://github.com/user-attachments/assets/838ba0b8-3d68-4963-8da9-05f596dfea85" />

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



````
## depositFor

### Pre-conditions (ДО)
| Invariant | Check | Status |
|-----------|-------|--------|
| rootToken != ETHER_ADDRESS | require ✅ | closed |
| user != address(0) | no check | ⚠️ → _depositFor |
| amount > 0 | no check | ⚠️ → _depositFor |
| balanceOf(user) >= amount | no check | ⚠️ → lockTokens |
| allowance(user, predicate) >= amount | no check | ⚠️ → lockTokens |

### Post-conditions (ПОСЛЕ)
| Invariant | Check | Status |
|-----------|-------|--------|
| _depositFor called with same args | in code ✅ | closed |

### Bugs
- none

### Centralization risks
- none
- 
---
---

### 2.RootChainManager.sol — `_depositFor`

 It performs deep security checks, locks the assets on Ethereum (L1), and triggers the minting process on Polygon (L2).

```solidity
function _depositFor(
    address user,              // L2 Recipient: Who gets the tokens on Polygon?
    address rootToken,         // L1 Token Address: The asset being bridged.
    bytes memory depositData   // Amount or Token ID.
) private {
    // [Check 1] Migration Status
    // Is bridging currently enabled for this specific token?
    if (migrationStatus[rootToken].isDepositDisabled) {
        revert("RootChainManager: DEPOSIT_DISABLED");
    }

    // [Check 2] Mapping Registry
    // Does this L1 token have a valid "partner" address on L2?
    bytes32 tokenType = tokenToType[rootToken];
    require(
        rootToChildToken[rootToken] != address(0x0) && tokenToType[rootToken] != 0,
        "RootChainManager: TOKEN_NOT_MAPPED"
    );

    // [Check 3] Predicate Check
    // Is there a logic handler (Predicate) for this specific token type?
    address predicateAddress = typeToPredicate[tokenType];
    require(predicateAddress != address(0), "RootChainManager: INVALID_TOKEN_TYPE");

    // [Check 4] Recipient Safety
    // Prevent sending funds to the "0x0" address (burning by mistake).
    require(user != address(0), "RootChainManager: INVALID_USER");

    // [Action 1] Asset Locking
    // Transfer tokens from User to the Bridge (Predicate) on L1.
    ITokenPredicate(predicateAddress).lockTokens(_msgSender(), user, rootToken, depositData);

    // [Action 2] State Sync (L2 Signal)
    // Send a secure message to Polygon validators to mint assets for 'user' on L2.
    bytes memory syncData = abi.encode(user, rootToken, depositData);
    _stateSender.syncState(childChainManagerAddress, abi.encode(DEPOSIT, syncData));
}
