# JinOS Security Model
## Non-Custodial by Design
JinOS is non-custodial. This means:
- JinOS never stores, sees, or transmits your private key
- Your BSC wallet is created and controlled through Privy MPC (Multi-Party Computation)
- Even if the JinOS servers were compromised, attackers cannot access your funds
- Only you can authorize transactions from your wallet
## How Privy MPC Works
Privy uses threshold cryptography (MPC) to split your private key into multiple secure shares:

Your private key
       ↓
Split into 3 shares:
  - Share 1: Stored by Privy (encrypted, HSM-protected)
  - Share 2: Stored in your browser (encrypted local storage)
  - Share 3: Stored encrypted in Privy's recovery system
To sign a transaction:
  2-of-3 shares required → your key is NEVER reconstructed in full

This means:
- Privy alone cannot move your funds (they only have 1 share)
- JinOS (the app) never has any share
- If you lose your device, you can recover via email + Privy recovery
For full Privy security details: https://docs.privy.io/guide/security
## Authentication
- Login: email OTP — no passwords stored anywhere
- Session token: Privy issues a JWT, your privyUserId is extracted
- API auth: All requests include Authorization: Bearer <privyUserId>
- Backend: Validates the bearer token format, scopes all DB queries to that user ID
- No cross-user data access is possible by design
## What the backend stores:
- Your Privy user ID (public identifier)
- Your email address (from Privy, optional)
- Your BSC wallet address (public, on-chain data)
- Virtual card metadata (card number, balance, status)
- Transaction history
- AI agent chat messages (for your session