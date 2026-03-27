# 📥 Deposit Flow: Technical Breakdown (L1 → L2)

This section provides a deep-dive analysis of the **Lock-and-Mint** mechanism. We trace the asset's path from the user's transaction on Ethereum to the state synchronization event.

---

## 🖼 Deposit Process Visualization

<img width="800" height="1024" alt="image" src="https://github.com/user-attachments/assets/838ba0b8-3d68-4963-8da9-05f596dfea85" />


### How It Works

**Step 1 — User initiates deposit on L1.**
The user calls `depositFor()` on RootChainManager, specifying the recipient address on L2, the token, and the amount.


#**Step 2 — Tokens are locked on L1.**
`lockTokens()` transfers the tokens from the user to the ERC20Predicate contract. The tokens are held in custody on Ethereum until the user withdraws.


#**Step 3 — Signal is sent to L2.**
`_depositFor()` calls StateSender which emits a sync event. Polygon validators pick up this event off-chain and relay it to L2.


#**Step 4 — L2 receives the signal.**
`onStateReceive()` on ChildChainManager is triggered by the validators. It decodes the message and routes it to `_syncDeposit()`.


#**Step 5 — Tokens are minted on L2.**
`_syncDeposit()` finds the corresponding L2 token address and calls `deposit()` on the ChildToken contract. The user receives the tokens on Polygon.

---
---
## 🔍 Line-by-Line Code Analysis


### 1. RootChainManager.sol — `depositFor`
This is the primary external gateway. It acts as a safety filter before passing the data to the internal engine.
---
<img width="800" height="1024" alt="image" src="https://github.com/user-attachments/assets/7ebf5e0b-5c29-405a-a890-34d4651af0ad" />



---

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
## Invariants

### Pre-conditions 
| Invariant | Check | Status |
|-----------|-------|--------|
| rootToken != ETHER_ADDRESS | require ✅ | closed |
| user != address(0) | no check | ⚠️ → _depositFor |
| amount > 0 | no check | ⚠️ → _depositFor |
| balanceOf(user) >= amount | no check | ⚠️ → lockTokens |
| allowance(user, predicate) >= amount | no check | ⚠️ → lockTokens |

### Post-conditions 
| Invariant | Check | Status |
|-----------|-------|--------|
| _depositFor called with same args | in code ✅ | closed |

### Bugs
- none

### Centralization risks
- none

---
---

### 2.RootChainManager.sol — `_depositFor`

It performs deep security checks, locks the assets on Ethereum (L1), and sends a signal to Polygon validators (off-chain) to mint the corresponding tokens on L2.
 
 ---
 
<img width="800" height="1024" alt="image" src="https://github.com/user-attachments/assets/982c8b6d-3126-4147-beb8-6e4c781748fa" />




---

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
````
## Invariants

### Pre-conditions 
| Invariant | Check | Status |
|-----------|-------|--------|
| user != address(0) | require ✅ | closed |
| amount > 0 | no check | ⚠️ → lockTokens |
| balanceOf(user) >= amount | no check | ⚠️ → lockTokens |
| allowance(user, predicate) >= amount | no check | ⚠️ → lockTokens |
| migrationStatus[rootToken].isDepositDisabled == false | check ✅ | closed |
| tokenToType[rootToken] != 0 | require ✅ | closed |
| rootToChildToken[rootToken] != address(0) | require ✅ | closed |
| typeToPredicate[tokenType] != address(0) | require ✅ | closed |

### Post-conditions 
| Invariant | Check | Status |
|-----------|-------|--------|
| balanceOf(predicate) += amount | safeTransferFrom ✅ | closed |
| balanceOf(user) -= amount | safeTransferFrom ✅ | closed |
| syncState sent with correct data | in code ✅ | closed |

### Bugs
- none

### Centralization risks
- MAPPER_ROLE can set malicious childToken
- ADMIN can replace StateSender
---
---

### 3.ERC20Predicate.sol — `lockTockens`
```solidity
function lockTokens(
    address depositor,       // The source address providing the funds
    address depositReceiver, // The destination address on L2
    address rootToken,       // The ERC20 token being bridged
    bytes calldata depositData // Encoded transfer parameters (amount)
)
    external
    override
    only(MANAGER_ROLE) // 🔐 Restricted to RootChainManager
{
    // [Action 1] Data Decoding
    // Unpacking the byte array into a uint256 amount.
    uint256 amount = abi.decode(depositData, (uint256));
    
    // [Action 2] Event Emission
    // Signaling the lock to off-chain relayers (StateSync).
    emit LockedERC20(depositor, depositReceiver, rootToken, amount);
    
    // [Action 3] Asset Seizure
    // Transfer tokens from User to the Predicate contract.
    IERC20(rootToken).safeTransferFrom(depositor, address(this), amount);
}
```
## lockTokens

### Pre-conditions
| Invariant | Check | Status |
|-----------|-------|--------|
| only MANAGER_ROLE | modifier ✅ | closed |
| amount > 0 | no check | ⚠️ → MEDIUM BUG |
| balanceOf(depositor) >= amount | safeTransferFrom ✅ | closed |
| allowance(depositor, predicate) >= amount | safeTransferFrom ✅ | closed |

### Post-conditions (ПОСЛЕ)
| Invariant | Check | Status |
|-----------|-------|--------|
| balanceOf(predicate) += amount | safeTransferFrom ✅ | closed |
| balanceOf(depositor) -= amount | safeTransferFrom ✅ | closed |
| LockedERC20 event emitted | in code ✅ | closed |

### Bugs
- [MEDIUM] amount > 0 not checked anywhere in deposit flow
  → zero deposit possible → griefing attack on validators

### Centralization risks
- none

  ---
  ---
  

# StateSender — Off-chain Bridge Component

StateSender is an off-chain component operated by Polygon validators. It listens for the `LockedERC20` event on L1 and relays the deposit signal to L2 by calling `onStateReceive` on ChildChainManager. Outside the audit scope.

---
---
- ### 5. ChildChainManager.sol — `onStateReceive`
This is the main entry point on the Polygon (L2) side. It acts as a "Postman" that receives messages from the Ethereum StateSender and routes them to the correct internal logic (Deposit or Token Mapping).

````solidity

function onStateReceive(uint256 /* stateId */, bytes calldata data)
    external 
    override
    only(STATE_SYNCER_ROLE) // [Check] Security: Only official Polygon validators/system can call this
{
    // [Action] First-Level Decoding
    // Unpacking the main "envelope" to identify the action type (syncType) 
    // and the specific content (syncData).
    (bytes32 syncType, bytes memory syncData) = abi.decode(
        data,
        (bytes32, bytes) // syncType is typically a keccak256 hash (e.g., DEPOSIT or MAP_TOKEN)
    );

    // [Logic] Routing Switch: Determining the action based on the hash
    if (syncType == DEPOSIT) {
        // [Action] Deposit Execution
        // If the hash matches 'DEPOSIT', it triggers the internal minting flow
        _syncDeposit(syncData);
        
    } else if (syncType == MAP_TOKEN) {
        // [Action] Second-Level Decoding (Registration)
        // Extracting L1/L2 token addresses and the mapping type from the payload
        (address rootToken, address childToken, ) = abi.decode(
            syncData,
            (address, address, bytes32)
        );
        
        // [Action] Permanent Mapping
        // Registers the link between the Ethereum (Root) and Polygon (Child) token addresses
        _mapToken(rootToken, childToken);
        
    } else {
        // [Check] Security Filter
        // Reverts if the syncType is unrecognized, protecting against corrupted or malicious data
        revert("ChildChainManager: INVALID_SYNC_TYPE");
    }
}

````
## Invariants

### Pre-conditions
| Invariant | Check | Status |
|-----------|-------|--------|
| only STATE_SYNCER_ROLE | modifier ✅ | closed |
| syncType is valid | revert ✅ | closed |
| data correctly encoded | abi.decode ✅ | closed |

### Post-conditions
| Invariant | Check | Status |
|-----------|-------|--------|
| _syncDeposit called if syncType == DEPOSIT | in code ✅ | closed |
| _mapToken called if syncType == MAP_TOKEN | in code ✅ | closed |

### Bugs
- none

### Centralization risks
- STATE_SYNCER_ROLE controlled by admin → admin can trigger fake deposits on L2(replay attack)

 ---
 ---
 
- ### 5. ChildChainManager.sol — `_syncDeposit`

This is the final step of the deposit flow on L2 (Polygon). It receives the validated signal from the Ethereum side and triggers the actual minting of assets.

```solidity

function _syncDeposit(bytes memory syncData) internal {
    // [Action 1] Data Decoding
    // Extracting the user address, the root token (L1), and the specific deposit payload.
    (address user, address rootToken, bytes memory depositData) = abi.decode(
        syncData,
        (address, address, bytes)
    );

    // [Action 2] Mapping Lookup
    // Finding the corresponding L2 token address for the provided L1 root token.
    address childTokenAddress = rootToChildToken[rootToken];

    // [Check] Integrity Guard
    // Ensuring the token is registered in the bridge system before proceeding.
    require(
        childTokenAddress != address(0),
        "ChildChainManager: TOKEN_NOT_MAPPED"
    );

    // [Action 3] Minting / Deposit Execution
    // Calling the .deposit() function on the ChildToken contract to credit the user.
    IChildToken(childTokenAddress).deposit(user, depositData);
}
```
## Invariants

### Pre-conditions
| Invariant | Check | Status |
|-----------|-------|--------|
| rootToChildToken[rootToken] != address(0) | require ✅ | closed |
| amount > 0 | no check | ⚠️ → deposit() |
| user != address(0) | no check | ⚠️ → deposit() |

### Post-conditions
| Invariant | Check | Status |
|-----------|-------|--------|
| balanceOf(user) on L2 += amount | deposit() ✅ | closed |
| correct childToken called | rootToChildToken ✅ | closed |

### Bugs
- none

### Centralization risks
- MAPPER_ROLE can set malicious childTokenAddress
````

---
---
