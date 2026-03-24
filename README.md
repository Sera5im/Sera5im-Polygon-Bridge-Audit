<p align="center">
  <img width="700" height="645" alt="image" src="https://github.com/user-attachments/assets/79e1fe3c-5476-4015-9445-b06ce1715b4f" />
</p>

# 🌉 Report #1: PoS Polygon Bridge Audit

Part of my **20 Bridges Series** for professional portfolio. This audit focuses strictly on the **Core Money Flow** and critical security constraints.

---

### 🛡 Audit Scope & Focus
Unlike a general code review, this audit ignores auxiliary boilerplate and standard libraries. I have specifically isolated and analyzed the **~25% of the codebase** that controls asset movement:
* **Critical Path:** Only functions directly moving, locking, or minting funds.
* **Security Layer:** Ownership, mapping, and predicate logic that can override the system.
* **Noise Filter:** Standard OZ libraries, mocks, and administrative helpers are excluded to focus on actual cross-chain risk.

---

## 🧭 Navigation

### 📥 [Deposit Flow (L1 → L2)](./DEPOSIT_DETAILS.md)
* Bridge schema and token path from `approve` to L2 minting.
* Token mapping and Predicate logic (entry control).

### 📤 [Withdrawal Flow (L2 → L1)](./WITHDRAWAL_DETAILS.md)
* L2 burn sequence and L1 `exit` unlock mechanism.
* Merkle Proof verification and Checkpoint Manager interaction.

### 🛠 [System & Admin Logic](./SYSTEM_DETAILS.md)
* Administrative functions, `onlyOwner` access, emergency pause, and upgrades.

---

## 🛠 Tech Stack
* **L1:** RootChainManager, ERC20Predicate, StateSender.
* **L2:** ChildChainManager, ChildToken.

---

> **Status:** 1/20 Analyzed. 
> **Focus:** Logic Integrity & Critical Function Security.
