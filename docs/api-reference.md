JinOS API Reference
Base URL (production): https://jinos.xyz/api
Base URL (development): http://localhost:<PORT>/api

All authenticated endpoints require the header:

Authorization: Bearer <privyUserId>

The privyUserId is the user's Privy user ID (format: did:privy:...).

Health
GET /health
Check API server status.

Response:

{
  "status": "ok",
  "ts": 1713000000000
}

Users
POST /users/sync
Create or update a user record from Privy auth. Called automatically on login.

Request body:

{
  "privyId": "did:privy:abc123",
  "email": "user@example.com",
  "embeddedWalletAddress": "0xabc..."
}

Response:

{
  "id": 1,
  "privyId": "did:privy:abc123",
  "email": "user@example.com",
  "embeddedWalletAddress": "0xabc...",
  "createdAt": "2025-01-01T00:00:00Z",
  "updatedAt": "2025-01-01T00:00:00Z"
}

Dashboard
GET /dashboard/summary
Get wallet balance, BSC network stats, and top market movers.

Auth: Required

Response:

{
  "totalBalanceUsd": 1234.56,
  "totalBalanceBnb": 2.0576,
  "change24hUsd": 0,
  "change24hPercent": 0,
  "gasPrice": 3,
  "gasPriceUnit": "Gwei",
  "bscBlockNumber": 48000000,
  "networkStatus": "online",
  "topMovers": [
    {
      "symbol": "BNB",
      "name": "BNB Chain",
      "price": 600.00,
      "change24h": 2.5,
      "volume24h": 1500000000,
      "marketCap": 90000000000,
      "logoUrl": "https://..."
    }
  ],
  "pendingTxCount": 0
}

GET /dashboard/activity
Get recent card transaction activity for the authenticated user.

Auth: Required

Response: Array of activity items:

[
  {
    "id": "42",
    "type": "receive",
    "label": "Funded Card",
    "detail": "50 USDT → $50.00",
    "timestamp": "2025-01-01T12:00:00Z",
    "status": "confirmed",
    "amountUsd": 50.00
  }
]

GET /dashboard/market
Get live market data for BSC ecosystem coins.

Auth: Not required

Response: Array of market coin objects (same shape as topMovers above).

Wallet
GET /wallet
Get wallet info and live token balances for the authenticated user.

Auth: Required

Response:

{
  "address": "0xabc...",
  "network": "BSC Mainnet",
  "chainId": 56,
  "balances": [
    {
      "symbol": "BNB",
      "name": "BNB",
      "balance": 2.0576,
      "balanceUsd": 1234.56,
      "price": 600.00,
      "priceChange24h": 2.5,
      "logoUrl": "https://...",
      "contractAddress": "native"
    }
  ],
  "totalUsd": 1234.56
}

Cards
GET /cards
List all virtual cards for the authenticated user.

Auth: Required

Response: Array of card objects:

[
  {
    "id": 1,
    "userId": "did:privy:abc123",
    "label": "Subscriptions",
    "cardNumber": "4000 1234 5678 9012",
    "expiry": "01/28",
    "cvv": "123",
    "balance": "150.000000",
    "spendingLimit": null,
    "status": "active",
    "type": "debit",
    "gradient": null,
    "createdAt": "2025-01-01T00:00:00Z",
    "updatedAt": "2025-01-01T00:00:00Z"
  }
]

POST /cards
Issue a new virtual card.

Auth: Required

Request body:

{
  "label": "Shopping",
  "type": "debit"
}

Response: Card object (same shape as above)

GET /cards/:id
Get a specific card including its transaction history.

Auth: Required
Params: id — card ID (integer)

Response:

{
  "card": { "...card fields..." },
  "transactions": [
    {
      "id": 1,
      "cardId": 1,
      "type": "fund",
      "merchant": null,
      "category": null,
      "amount": "50.000000",
      "fromToken": "USDT",
      "fromAmount": "50.00000000",
      "fee": null,
      "txHash": null,
      "note": null,
      "createdAt": "2025-01-01T00:00:00Z"
    }
  ]
}

PATCH /cards/:id
Update card status or spending limit.

Auth: Required

Request body (any combination):

{
  "status": "frozen",
  "spendingLimit": "500.00"
}

Response: Updated card object

POST /cards/:id/fund
Fund a card from the user's wallet balance.

Auth: Required

Request body:

{
  "amount": "100.00",
  "fromToken": "USDT"
}

Response:

{
  "card": { "...updated card..." },
  "transaction": { "...card transaction record..." }
}

Transactions
GET /transactions
Get on-chain transaction history.

Auth: Required

Query params:

Param	Type	Default	Description
limit	integer	20	Results per page
offset	integer	0	Pagination offset
type	string	all	Filter: send, receive, swap, stake, all
Response:

{
  "transactions": [
    {
      "id": 1,
      "hash": "0xabc...",
      "type": "receive",
      "status": "confirmed",
      "fromAddress": "0xsender...",
      "toAddress": "0xrecipient...",
      "tokenSymbol": "BNB",
      "tokenSymbolTo": null,
      "amount": "1.50000000",
      "amountUsd": "900.00",
      "gasFee": "0.00010000",
      "gasFeeUsd": "0.0600",
      "blockNumber": 48000000,
      "createdAt": "2025-01-01T00:00:00Z"
    }
  ],
  "total": 42
}

Portfolio
GET /portfolio
Get portfolio chart data and asset allocation.

Auth: Required

Response:

{
  "totalUsd": 1234.56,
  "change24hUsd": 25.00,
  "change24hPercent": 2.07,
  "chartData": [
    { "date": "2025-01-01", "value": 1200.00 },
    { "date": "2025-01-02", "value": 1234.56 }
  ],
  "assets": [
    {
      "symbol": "BNB",
      "name": "BNB",
      "allocation": 85.5,
      "balanceUsd": 1055.56,
      "balance": 1.759,
      "price": 600.00,
      "change24h": 2.5,
      "logoUrl": "https://..."
    }
  ]
}

Agent
POST /agent/chat
Send a message to the AI financial agent.

Auth: Required

Request body:

{
  "message": "What is my portfolio allocation?"
}

Response:

{
  "id": 10,
  "sessionId": "default",
  "message": "Your portfolio is 85% BNB and 15% USDT...",
  "role": "assistant",
  "actions": null,
  "createdAt": "2025-01-01T12:00:00Z"
}

GET /agent/history
Get chat history for the authenticated user.

Auth: Required

Response: Array of message objects (same shape as above, role is user or assistant)

GET /agent/config
Get agent task configuration.

Auth: Required

Response:

{
  "model": "gpt-4o-mini",
  "temperature": 0.7,
  "tasks": [
    {
      "id": "auto-rebalance",
      "name": "Auto Rebalance",
      "enabled": false,
      "description": "Automatically rebalance your portfolio based on target allocations"
    },
    {
      "id": "yield-optimizer",
      "name": "Yield Optimizer",
      "enabled": false,
      "description": "Move idle assets into yield-generating protocols"
    },
    {
      "id": "spend-monitor",
      "name": "Spend Monitor",
      "enabled": true,
      "description": "Alert when card spending exceeds daily/weekly thresholds"
    },
    {
      "id": "auto-topup",
      "name": "Auto Top-Up Card",
      "enabled": false,
      "description": "Automatically fund card when balance drops below minimum"
    }
  ]
}

PUT /agent/config
Update agent task configuration.

Auth: Required

Request body:

{
  "tasks": [
    { "id": "auto-rebalance", "enabled": true },
    { "id": "spend-monitor", "enabled": false }
  ]
}

Response: Updated config object (same shape as GET)

Swap
GET /swap
Swap endpoint placeholder.

Response:

{
  "message": "Swap coming soon",
  "supported": false
}

Error Responses
All errors follow this format:

{
  "error": "Human-readable error message"
}

Status	Meaning
400	Bad request — missing or invalid parameters
401	Unauthorized — missing or invalid Bearer token
403	Forbidden — authenticated but not authorized for this resource
404	Not found
500	Internal server error