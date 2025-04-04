# Guard Protocol: Decentralized Social Recovery for Crypto Assets

**Written by Amschel Kariuki**

**Version:** 0.1 (MVP Concept)
**Date:** April 04, 2025

## Abstract

Loss of private keys represents one of the most significant barriers to mainstream cryptocurrency adoption and a source of substantial financial loss for existing users. Current solutions often involve trade-offs between security, usability, and true self-custody. Guard Protocol introduces a novel approach leveraging Shamir's Secret Sharing (SSS) for key sharding, client-side cryptography for security, trusted social circles (friends/family) for shard custody, and the Internet Computer (ICP) blockchain for decentralized, automated backend logic and encrypted shard storage facilitation. Users generate keys client-side, split them into encrypted shards distributed to designated guardians via Telegram interactions mediated by an ICP canister, and can recover keys by retrieving a threshold number of shards through the same mechanism, ensuring the user remains the sole custodian of their keys throughout the process.

## 1. Introduction: The Private Key Problem

Cryptocurrency ownership is defined by control over private keys. The adage "not your keys, not your coins" underscores the importance of secure key management. However, existing solutions present challenges:

- **Self-Custody (Manual Backups):** Writing down seed phrases is simple but vulnerable to physical loss, theft, damage, or discovery.
- **Hardware Wallets:** Improve security but are subject to loss, theft, damage, supply chain attacks, and can be a single point of failure if the device and its backup are compromised simultaneously.
- **Centralized Custodians:** Offer convenience but require trusting a third party, relinquishing true ownership, and are vulnerable to hacks, insider threats, and censorship.
- **Multi-Sig Wallets:** Provide redundancy but often require technical expertise, on-chain fees for setup/transactions, and don't typically protect the underlying seed phrase itself.
- **Social Recovery (Existing):** Often relies on centralized servers, complex smart contract interactions, or insecure communication channels.

There is a clear need for a user-friendly, secure, and decentralized key recovery mechanism that preserves self-custody.

## 2. The Guard Protocol Solution

Guard Protocol combines established cryptographic techniques with modern decentralized infrastructure to offer robust social recovery:

- **Client-Side Operations:** Key generation, SSS splitting, shard encryption, shard decryption, and key reconstruction _always_ occur on the user's local device (e.g., in-browser or via an extension). The unencrypted private key or seed phrase never leaves the user's control.
- **Shamir's Secret Sharing (SSS):** A proven cryptographic algorithm where a secret (the private key/seed) is split into `n` unique shards, such that any `k` shards (where `k <= n`) can reconstruct the original secret, but `k-1` shards reveal no information.
- **Trusted Guardian Network:** Users designate individuals they already trust (friends, family) as guardians. This leverages existing social trust rather than relying on anonymous or pseudo-anonymous actors.
- **End-to-End Encryption:** Shards are individually encrypted client-side using a user-defined secret (e.g., derived from a password) before transmission or storage.
- **Decentralized Backend (ICP):** An Internet Computer canister orchestrates the _communication_ and _storage facilitation_ process. It handles guardian notifications via Telegram (using HTTPS outcalls), receives guardian approvals, stores the _user-encrypted_ shards, and serves them back to the user during recovery. The canister _never_ has access to unencrypted shards or the user's encryption secret.
- **Simplified User Experience:** Interaction primarily occurs through a simple client-side interface and familiar Telegram messages for guardians, abstracting the underlying complexity.

## 3. Architecture

Guard Protocol consists of three main components:

### 3.1. Client-Side Application

- **Implementation:** Web application (JavaScript) or Browser Extension.
- **Responsibilities:**
  - User Interface (UI/UX) for setup and recovery.
  - Secure generation of BIP-39 mnemonic seeds or private keys.
  - Implementation of a standard SSS library (e.g., `k`-of-`n` threshold).
  - Derivation of a strong encryption key from a user-provided password/phrase (using PBKDF2/Argon2).
  - AES-GCM (or similar standard) encryption/decryption of individual shards using the derived key.
  - Interaction with the ICP Guard Canister (via `agent-js` or similar) to send encrypted shards and recovery requests.
  - Receiving encrypted shards back from the canister during recovery.
  - Reconstruction of the original key/seed from decrypted shards.
  - Secure local storage of non-sensitive metadata (e.g., guardian identifiers, `k`, `n` values).

### 3.2. ICP Guard Canister

- **Implementation:** Motoko or Rust canister running on the Internet Computer.
- **Responsibilities:**
  - Exposes functions callable by the client application (e.g., `store_encrypted_shard`, `request_recovery`, `confirm_guardian_approval`).
  - Manages state securely: Mapping `(user_id, guardian_id)` to `encrypted_shard_blob`, tracking recovery request status.
  - Makes HTTPS outcalls to the Telegram Bot API to send notifications and interactive messages (buttons) to guardians.
  - Receives confirmations/approvals (likely via a secure webhook relayed to a canister function).
  - Stores encrypted shard blobs in stable storage.
  - Enforces basic logic (e.g., ensuring recovery approval before returning a shard).
  - **Crucially:** Handles only _encrypted_ data provided by the user client. Has no knowledge of decryption keys or original secrets.
  - Cycle Management: Requires adequate cycles for computation and storage.

### 3.3. Telegram Bot Interface

- **Implementation:** Standard Telegram Bot.
- **Responsibilities:**
  - Acts as a communication relay between the ICP Guard Canister and the human guardians.
  - Receives instructions from the canister (via HTTPS outcalls) to message specific guardians.
  - Displays user-friendly messages and interactive buttons ("Accept Shard Storage", "Approve Recovery").
  - Sends guardian responses back to the ICP Canister (via secure webhook/API call).
  - Requires secure management of the Bot API Token.

## 4. User Journey

### 4.1. Setup Phase

1.  **Initiation:** User opens the Guard client application.
2.  **Key Generation:** User generates a new key/seed (client-side). _Recommended: User makes a separate primary backup._
3.  **SSS Configuration:** User selects threshold `k` and total shards `n`.
4.  **Encryption Setup:** User provides a strong password/phrase for shard encryption. Key derived client-side.
5.  **Guardian Designation:** User identifies `n` trusted guardians (e.g., by their Telegram usernames).
6.  **Shard Generation & Encryption:** Client app splits key using SSS into `n` shards and encrypts each shard individually.
7.  **Distribution via ICP:** Client calls `store_encrypted_shard` on the ICP canister for each guardian/encrypted shard pair.
8.  **Guardian Notification:** Canister uses Telegram bot to notify each guardian, requesting confirmation to store the encrypted shard.
9.  **Guardian Confirmation:** Guardian interacts with the Telegram bot to accept.
10. **Storage Confirmation:** Bot informs canister; canister stores the encrypted shard persistently. Client app is notified of success.

### 4.2. Recovery Phase

1.  **Initiation:** User opens Guard client app, chooses "Recover Key".
2.  **Decryption Setup:** User enters their shard encryption password/phrase.
3.  **Shard Request:** Client app determines which guardians are needed (at least `k`) and calls `request_recovery` on the ICP canister for each required guardian.
4.  **Guardian Notification:** Canister uses Telegram bot to notify relevant guardians, requesting approval to release their shard.
5.  **Guardian Approval:** Guardian interacts with the Telegram bot to approve the release.
6.  **Shard Retrieval:** Bot informs canister; canister marks shard as approved for retrieval. Client app polls or is notified, then calls a function (e.g., `get_approved_shard`) on the canister to fetch the _encrypted_ shard.
7.  **Decryption & Reconstruction:** Client app receives at least `k` encrypted shards, decrypts them locally using the user's password/phrase, and uses the SSS library to reconstruct the original key/seed.
8.  **Key Displayed:** The recovered key/seed is displayed to the user within the client application.

## 5. Security Model & Considerations

- **Self-Custody Preserved:** The user is always in control of the key generation and reconstruction. The protocol facilitates backup and recovery, not custody.
- **Client-Side Encryption:** The cornerstone of security. Shards are encrypted _before_ leaving the user's device. Compromise of the backend (ICP canister or Telegram bot) does not reveal the user's key, only encrypted blobs.
- **SSS Threshold:** `k-1` compromised guardians cannot reconstruct the key. Collusion among `k` or more guardians _is_ a theoretical risk, mitigated by the user choosing _trusted_ individuals unlikely to collude.
- **Encryption Key Security:** The user's shard encryption password/phrase becomes a critical secret. If lost, recovery is impossible. If weak, it compromises shard security. Robust key derivation (Argon2/PBKDF2) is essential.
- **Guardian Reliability:** Users must choose guardians who are likely to remain accessible, retain access to their Telegram account, and be responsive during recovery. The protocol cannot fully mitigate human factors. Recommend `n > k` (e.g., 3 of 5) for redundancy.
- **ICP Canister Security:** While it doesn't hold keys, the canister must be secured against denial-of-service, unauthorized state changes, or bugs in logic (e.g., releasing shards without proper approval). Cycle management is critical for liveness.
- **Telegram Security:** Relies on the security of Telegram accounts and the Bot API. Guardian account compromise could lead to unauthorized recovery approval (mitigated if the user requires multiple approvals).
- **Implementation Vulnerabilities:** Standard software vulnerabilities (XSS in web app, bugs in crypto implementation, insecure canister code) must be mitigated through careful development and auditing.

## 6. Future Roadmap / Potential Enhancements

- **Guardian Rotation/Revocation:** Mechanism to replace unresponsive or untrusted guardians.
- **Multi-Device Support:** Synchronizing encrypted metadata across user devices.
- **Additional Communication Channels:** Integrating SMS (via Twilio), Signal, etc., for guardian notifications.
- **Optional Professional Guardians:** Introducing vetted, potentially insured third-party guardians as an option (likely requiring a different economic model).
- **Time Locks/Delays:** Adding optional delays to recovery requests to allow users time to react if a request was malicious.
- **Formal Security Audits:** Commissioning third-party audits of client-side code, canister code, and overall architecture.
- **Gasless Transactions (ICP):** Exploring ways to subsidize or abstract cycle costs for users.

## 7. Conclusion

Guard Protocol offers a pragmatic and secure approach to crypto asset recovery by combining the mathematical robustness of SSS, the security of client-side encryption, the inherent value of existing social trust networks, and the decentralized infrastructure of the Internet Computer. By automating the storage and retrieval of _encrypted_ shards via ICP and Telegram, it significantly improves usability over manual methods while maintaining user self-custody, addressing a critical pain point in the cryptocurrency ecosystem.
