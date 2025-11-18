# API Requirements Specification

## Document Information

**API Name:** Cross-Chain Bridge API  
**Version:** 2.0  
**Date:** September 2023  
**Author:** Oleh Syzonov, Lead Business Analyst  
**Project:** Blockchain Infrastructure Platform  
**Status:** Approved for Development  
**Base URL:** `https://api.bridge.platform.example` 🔔 **ANONYMIZED**

---

## Overview

### Purpose
This document specifies the RESTful API requirements for the Cross-Chain Bridge service, enabling institutional users to transfer digital assets between Ethereum and Solana blockchains.

### Audience
- Backend developers implementing the API
- Frontend developers consuming the API
- QA engineers testing the API
- Integration partners connecting to the API

### API Design Principles
- **RESTful:** Follows REST architectural constraints
- **Resource-Oriented:** URLs represent resources, not actions
- **Stateless:** Each request contains all necessary information
- **Versioned:** API version in URL path for backward compatibility
- **Secure:** All endpoints require authentication via OAuth 2.0
- **Idempotent:** POST/PUT operations use idempotency keys to prevent duplicates

---

## Authentication

### OAuth 2.0 Client Credentials Flow

**Token Endpoint:** `POST /oauth/token`

**Request:**
```http
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={YOUR_CLIENT_ID}
&client_secret={YOUR_CLIENT_SECRET}
&scope=bridge:read bridge:write
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "bridge:read bridge:write"
}
```

**Requirements:**
- ✅ Access tokens expire after 1 hour
- ✅ Refresh tokens valid for 30 days
- ✅ All subsequent API calls include: `Authorization: Bearer {access_token}`
- ✅ Tokens are JWT format with user/client claims
- ✅ Rate limit: 100 requests per minute per client

---

## API Endpoints

### 1. Initiate Transfer

**Endpoint:** `POST /v2/transfers`

**Description:**  
Initiates a new cross-chain asset transfer from source chain to destination chain.

**Business Rules:**
- User must be KYC-verified
- Transfer amount must be ≥ $100 (minimum threshold)
- Transfer amount must be ≤ $5,000,000 (per-transaction limit)
- User must have sufficient balance + gas fees
- Daily transfer limit: $10,000,000 per user
- Transfers > $100,000 require compliance approval

**Request Headers:**
```http
POST /v2/transfers
Authorization: Bearer {access_token}
Content-Type: application/json
Idempotency-Key: {unique-uuid-v4}
X-Request-ID: {unique-request-id}
```

**Request Body:**
```json
{
  "source_chain": "ETHEREUM",
  "destination_chain": "SOLANA",
  "asset": {
    "symbol": "USDC",
    "contract_address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
  },
  "amount": "50000.000000",
  "destination_address": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "metadata": {
    "reference_id": "CLIENT-TXN-12345",
    "notes": "Q1 liquidity transfer"
  }
}
```

**Field Specifications:**

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `source_chain` | string | Yes | Enum: ["ETHEREUM", "SOLANA"] | Source blockchain |
| `destination_chain` | string | Yes | Enum: ["ETHEREUM", "SOLANA", "BSC"] | Destination blockchain |
| `asset.symbol` | string | Yes | Must be whitelisted asset | Token symbol (e.g., "USDC") |
| `asset.contract_address` | string | Yes | Valid contract address format | Token contract address |
| `amount` | string | Yes | Decimal with 6 decimals, >=$100, <=$5M | Transfer amount |
| `destination_address` | string | Yes | Valid address for destination chain | Recipient wallet address |
| `metadata.reference_id` | string | No | Max 100 chars | Client's internal reference |
| `metadata.notes` | string | No | Max 500 chars | Transfer notes |

**Response (202 Accepted):**
```json
{
  "transfer_id": "TXN-2023-09-15-ABCD1234",
  "status": "INITIATED",
  "source_chain": "ETHEREUM",
  "destination_chain": "SOLANA",
  "asset": {
    "symbol": "USDC",
    "contract_address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "amount": "50000.000000"
  },
  "fees": {
    "bridge_fee": {
      "amount": "5.000000",
      "asset": "USDC"
    },
    "gas_fee_estimate": {
      "amount": "0.015",
      "asset": "ETH",
      "usd_estimate": "45.00"
    }
  },
  "net_amount": "49995.000000",
  "estimated_completion_time": "2023-09-15T14:45:00Z",
  "estimated_duration_seconds": 600,
  "created_at": "2023-09-15T14:35:00Z",
  "links": {
    "self": "/v2/transfers/TXN-2023-09-15-ABCD1234",
    "status": "/v2/transfers/TXN-2023-09-15-ABCD1234/status"
  }
}
```

**Error Responses:**

**400 Bad Request - Validation Error:**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "amount",
        "error": "Amount must be at least $100. Provided: $50"
      }
    ]
  },
  "request_id": "req_abc123"
}
```

**400 Bad Request - Insufficient Balance:**
```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Insufficient balance for transfer",
    "details": {
      "available_balance": "25000.000000",
      "required_balance": "50015.000000",
      "shortfall": "25015.000000"
    }
  },
  "request_id": "req_abc123"
}
```

**429 Too Many Requests - Rate Limit:**
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Daily transfer limit exceeded",
    "details": {
      "daily_limit": "10000000.00",
      "used_today": "9800000.00",
      "remaining": "200000.00",
      "resets_at": "2023-09-16T00:00:00Z"
    }
  },
  "request_id": "req_abc123"
}
```

**503 Service Unavailable - Bridge Maintenance:**
```json
{
  "error": {
    "code": "BRIDGE_MAINTENANCE",
    "message": "Bridge is temporarily unavailable for maintenance",
    "details": {
      "estimated_restore_time": "2023-09-15T16:00:00Z",
      "maintenance_reason": "Scheduled upgrade"
    }
  },
  "request_id": "req_abc123"
}
```

---

### 2. Get Transfer Status

**Endpoint:** `GET /v2/transfers/{transfer_id}`

**Description:**  
Retrieves the current status and details of a transfer.

**Request:**
```http
GET /v2/transfers/TXN-2023-09-15-ABCD1234
Authorization: Bearer {access_token}
```

**Response (200 OK):**
```json
{
  "transfer_id": "TXN-2023-09-15-ABCD1234",
  "status": "COMPLETED",
  "source_chain": "ETHEREUM",
  "destination_chain": "SOLANA",
  "asset": {
    "symbol": "USDC",
    "amount": "50000.000000"
  },
  "destination_address": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "fees": {
    "bridge_fee": "5.000000",
    "gas_fee_actual": "0.0142"
  },
  "net_amount": "49995.000000",
  "timeline": {
    "initiated_at": "2023-09-15T14:35:00Z",
    "locked_at": "2023-09-15T14:36:15Z",
    "attested_at": "2023-09-15T14:39:20Z",
    "minted_at": "2023-09-15T14:40:45Z",
    "completed_at": "2023-09-15T14:40:50Z",
    "duration_seconds": 350
  },
  "transactions": {
    "source": {
      "chain": "ETHEREUM",
      "tx_hash": "0xabc123def456...",
      "block_number": 18234567,
      "confirmations": 25,
      "explorer_url": "https://etherscan.io/tx/0xabc123def456..."
    },
    "destination": {
      "chain": "SOLANA",
      "tx_hash": "xyz789abc123...",
      "slot": 234567890,
      "explorer_url": "https://solscan.io/tx/xyz789abc123..."
    }
  },
  "oracle_attestations": {
    "required": 5,
    "received": 7,
    "validators": [
      "oracle-1.example.com",
      "oracle-2.example.com",
      "oracle-3.example.com"
    ]
  },
  "metadata": {
    "reference_id": "CLIENT-TXN-12345",
    "notes": "Q1 liquidity transfer"
  }
}
```

**Transfer Status Values:**

| Status | Description | Next State(s) |
|--------|-------------|---------------|
| `INITIATED` | Transfer created, validations passed | `LOCKED`, `FAILED` |
| `LOCKED` | Assets locked on source chain | `ATTESTED`, `FAILED` |
| `ATTESTED` | Oracle consensus reached | `MINTED`, `FAILED` |
| `MINTED` | Assets minted on destination chain | `COMPLETED` |
| `COMPLETED` | Transfer successfully finished | (terminal state) |
| `FAILED` | Transfer failed at some step | (terminal state) |
| `CANCELLED` | User cancelled before lock | (terminal state) |
| `PENDING_COMPLIANCE` | Awaiting compliance approval | `INITIATED`, `REJECTED` |
| `REJECTED` | Compliance rejected transfer | (terminal state) |

---

### 3. List Transfers

**Endpoint:** `GET /v2/transfers`

**Description:**  
Retrieves a paginated list of transfers for the authenticated user.

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `status` | string | No | (all) | Filter by status (comma-separated for multiple) |
| `source_chain` | string | No | (all) | Filter by source blockchain |
| `destination_chain` | string | No | (all) | Filter by destination blockchain |
| `asset` | string | No | (all) | Filter by asset symbol |
| `from_date` | string | No | 30 days ago | ISO 8601 date (start of range) |
| `to_date` | string | No | now | ISO 8601 date (end of range) |
| `limit` | integer | No | 20 | Results per page (max: 100) |
| `offset` | integer | No | 0 | Pagination offset |
| `sort` | string | No | `-created_at` | Sort field (prefix `-` for descending) |

**Request:**
```http
GET /v2/transfers?status=COMPLETED,FAILED&limit=10&sort=-created_at
Authorization: Bearer {access_token}
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "transfer_id": "TXN-2023-09-15-ABCD1234",
      "status": "COMPLETED",
      "source_chain": "ETHEREUM",
      "destination_chain": "SOLANA",
      "asset": {
        "symbol": "USDC",
        "amount": "50000.000000"
      },
      "created_at": "2023-09-15T14:35:00Z",
      "completed_at": "2023-09-15T14:40:50Z"
    },
    {
      "transfer_id": "TXN-2023-09-14-XYZ9876",
      "status": "FAILED",
      "source_chain": "ETHEREUM",
      "destination_chain": "SOLANA",
      "asset": {
        "symbol": "USDT",
        "amount": "10000.000000"
      },
      "created_at": "2023-09-14T10:20:00Z",
      "failed_at": "2023-09-14T10:22:30Z",
      "failure_reason": "ORACLE_CONSENSUS_TIMEOUT"
    }
  ],
  "pagination": {
    "total": 145,
    "limit": 10,
    "offset": 0,
    "has_more": true
  },
  "links": {
    "self": "/v2/transfers?status=COMPLETED,FAILED&limit=10&offset=0",
    "next": "/v2/transfers?status=COMPLETED,FAILED&limit=10&offset=10",
    "last": "/v2/transfers?status=COMPLETED,FAILED&limit=10&offset=140"
  }
}
```

---

### 4. Get Supported Assets

**Endpoint:** `GET /v2/assets`

**Description:**  
Returns list of assets supported for cross-chain transfers with their configurations.

**Request:**
```http
GET /v2/assets
Authorization: Bearer {access_token}
```

**Response (200 OK):**
```json
{
  "assets": [
    {
      "symbol": "USDC",
      "name": "USD Coin",
      "supported_chains": ["ETHEREUM", "SOLANA", "BSC"],
      "min_transfer_amount": "100.000000",
      "max_transfer_amount": "5000000.000000",
      "decimals": 6,
      "contracts": {
        "ETHEREUM": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
        "SOLANA": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
      },
      "bridge_fee": "0.01",
      "bridge_fee_unit": "percentage",
      "min_bridge_fee": "5.000000",
      "max_bridge_fee": "500.000000"
    },
    {
      "symbol": "USDT",
      "name": "Tether USD",
      "supported_chains": ["ETHEREUM", "SOLANA"],
      "min_transfer_amount": "100.000000",
      "max_transfer_amount": "5000000.000000",
      "decimals": 6,
      "contracts": {
        "ETHEREUM": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
        "SOLANA": "Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB"
      },
      "bridge_fee": "0.01",
      "bridge_fee_unit": "percentage"
    }
  ]
}
```

---

### 5. Estimate Transfer Fees

**Endpoint:** `POST /v2/transfers/estimate`

**Description:**  
Estimates fees for a transfer without actually initiating it. Useful for showing users cost before commitment.

**Request:**
```http
POST /v2/transfers/estimate
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "source_chain": "ETHEREUM",
  "destination_chain": "SOLANA",
  "asset": "USDC",
  "amount": "50000.000000"
}
```

**Response (200 OK):**
```json
{
  "estimate": {
    "amount": "50000.000000",
    "bridge_fee": {
      "amount": "5.000000",
      "percentage": "0.01"
    },
    "gas_fee": {
      "amount": "0.015",
      "asset": "ETH",
      "usd_estimate": "45.00",
      "price_source": "Chainlink Oracle",
      "price_timestamp": "2023-09-15T14:30:00Z"
    },
    "net_amount": "49995.000000",
    "total_cost_usd": "50.00"
  },
  "estimated_duration_seconds": 600,
  "warnings": [
    {
      "code": "HIGH_GAS_PRICE",
      "message": "Current Ethereum gas prices are elevated. Consider waiting for lower gas prices."
    }
  ],
  "valid_until": "2023-09-15T14:45:00Z"
}
```

---

### 6. Cancel Transfer

**Endpoint:** `DELETE /v2/transfers/{transfer_id}`

**Description:**  
Attempts to cancel a transfer. Only possible if transfer status is `INITIATED` or `PENDING_COMPLIANCE`.

**Request:**
```http
DELETE /v2/transfers/TXN-2023-09-15-ABCD1234
Authorization: Bearer {access_token}
```

**Response (200 OK - Successfully Cancelled):**
```json
{
  "transfer_id": "TXN-2023-09-15-ABCD1234",
  "status": "CANCELLED",
  "cancelled_at": "2023-09-15T14:37:00Z",
  "message": "Transfer successfully cancelled"
}
```

**Response (409 Conflict - Cannot Cancel):**
```json
{
  "error": {
    "code": "CANNOT_CANCEL",
    "message": "Transfer cannot be cancelled in current status",
    "details": {
      "current_status": "LOCKED",
      "reason": "Assets have already been locked on source chain. Cancellation not possible."
    }
  }
}
```

---

## Webhooks

### Webhook Configuration

Users can configure webhook URLs to receive real-time notifications about transfer status changes.

**Endpoint:** `POST /v2/webhooks`

**Request:**
```json
{
  "url": "https://your-api.example.com/webhooks/bridge",
  "events": [
    "transfer.initiated",
    "transfer.locked",
    "transfer.attested",
    "transfer.completed",
    "transfer.failed"
  ],
  "secret": "your_webhook_secret_for_signature_verification"
}
```

### Webhook Payload Example

**Event:** `transfer.completed`

```json
{
  "event": "transfer.completed",
  "timestamp": "2023-09-15T14:40:50Z",
  "data": {
    "transfer_id": "TXN-2023-09-15-ABCD1234",
    "status": "COMPLETED",
    "source_chain": "ETHEREUM",
    "destination_chain": "SOLANA",
    "amount": "50000.000000",
    "asset": "USDC",
    "transactions": {
      "source_tx": "0xabc123...",
      "destination_tx": "xyz789..."
    }
  }
}
```

**Webhook Signature:**
- Header: `X-Signature: sha256=<hmac_signature>`
- HMAC-SHA256 of request body using webhook secret
- Recipients should verify signature to ensure authenticity

---

## Rate Limits

### API Rate Limits

| Endpoint | Limit | Window |
|----------|-------|--------|
| `POST /v2/transfers` | 10 requests | per minute |
| `GET /v2/transfers/{id}` | 100 requests | per minute |
| `GET /v2/transfers` | 30 requests | per minute |
| `GET /v2/assets` | 100 requests | per minute |
| All other endpoints | 60 requests | per minute |

**Rate Limit Headers:**
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1694790000
```

**429 Too Many Requests Response:**
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "API rate limit exceeded",
    "retry_after": 45
  }
}
```

---

## Error Codes Reference

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `VALIDATION_ERROR` | 400 | Request validation failed |
| `INSUFFICIENT_BALANCE` | 400 | User has insufficient balance |
| `UNSUPPORTED_ASSET` | 400 | Asset not supported for transfer |
| `UNSUPPORTED_CHAIN` | 400 | Blockchain not supported |
| `INVALID_ADDRESS` | 400 | Destination address invalid |
| `AMOUNT_TOO_LOW` | 400 | Transfer amount below minimum |
| `AMOUNT_TOO_HIGH` | 400 | Transfer amount exceeds maximum |
| `UNAUTHORIZED` | 401 | Invalid or expired access token |
| `FORBIDDEN` | 403 | User not authorized for this action |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | Request conflicts with current state |
| `RATE_LIMIT_EXCEEDED` | 429 | Rate limit exceeded |
| `BRIDGE_MAINTENANCE` | 503 | Bridge temporarily unavailable |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## Non-Functional Requirements

### Performance Requirements
- **Response Time:** 95th percentile < 500ms for all GET requests
- **Response Time:** 95th percentile < 1 second for POST requests
- **Throughput:** Support 100 concurrent requests minimum
- **Availability:** 99.9% uptime (excluding planned maintenance)

### Security Requirements
- **Authentication:** OAuth 2.0 with JWT tokens
- **Authorization:** Role-based access control (RBAC)
- **Encryption:** TLS 1.3 for all API traffic
- **API Keys:** Rotatable, revocable client credentials
- **Audit Logging:** All API calls logged with request/response

### Data Requirements
- **Idempotency:** POST requests support idempotency keys (24-hour window)
- **Consistency:** Strong consistency for transfer status
- **Retention:** API logs retained for 90 days minimum

---

## Testing Requirements

### Test Scenarios

**TS-001: Successful Transfer Flow**
1. Initiate transfer → 202 Accepted
2. Poll status → INITIATED → LOCKED → ATTESTED → MINTED → COMPLETED
3. Verify blockchain transactions on both chains

**TS-002: Insufficient Balance**
1. Initiate transfer with insufficient balance → 400 Bad Request
2. Error code: `INSUFFICIENT_BALANCE`

**TS-003: Rate Limit Enforcement**
1. Make 11 transfer requests in 60 seconds
2. 11th request → 429 Too Many Requests

**TS-004: Invalid Authentication**
1. Call API with expired token → 401 Unauthorized
2. Call API with no token → 401 Unauthorized

**TS-005: Idempotency**
1. Initiate transfer with idempotency key
2. Repeat same request with same key → Returns same transfer_id
3. No duplicate transfer created

### Test Data Requirements

🔔 **TODO:** Provide test API credentials and sandbox environment details

**Test Environment:**
- Base URL: `https://api-sandbox.bridge.platform.example`
- Test Ethereum network: Goerli testnet
- Test Solana network: Devnet
- Test tokens: Available from faucets

---

## Open Issues & Future Enhancements

❓ **ISSUE-001:** Should we support batch transfers (multiple assets in one request)?  
- **Status:** Awaiting product decision  
- **Impact:** Medium - Would reduce API calls for high-volume users

❓ **ISSUE-002:** GraphQL endpoint as alternative to REST?  
- **Status:** Under consideration  
- **Impact:** Low - Nice to have for complex queries

**Future Enhancement:** Add `GET /v2/transfers/{id}/timeline` endpoint for detailed status history

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Aug 2023 | O. Syzonov | Initial API specification |
| 1.5 | Sep 2023 | O. Syzonov | Added webhooks and rate limits |
| 2.0 | Sep 2023 | O. Syzonov | Added estimate endpoint and expanded error codes |

---

**Document Classification:** Internal - Technical Specification  
**Portfolio Note:** This API specification was created for blockchain bridge project (2023-2024). Demonstrates comprehensive API requirements documentation including authentication, endpoints, error handling, and non-functional requirements.

**Author Note:** This API design balanced institutional requirements (compliance, audit trails) with developer experience (clear errors, idempotency, webhooks). Successfully implemented and serving production traffic.
