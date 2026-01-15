# BitPay Tags - Decentralized Payment Requests on Stacks

A trustless payment request system enabling users to create, share, and fulfill Bitcoin-backed payment requests with built-in expiration and state management on the Stacks blockchain.

## Overview

BitPay Tags revolutionizes peer-to-peer payments by creating shareable payment requests that leverage sBTC on the Stacks blockchain. Users can generate tagged payment requests with custom amounts, expiration times, and memos, while payers can fulfill these requests seamlessly. Perfect for merchants, freelancers, and anyone needing a professional Bitcoin payment solution with guaranteed settlement.

## Key Features

- **Timestamped Payment Requests**: Create payment requests with automatic expiration
- **Decentralized Fulfillment**: Uses sBTC tokens for settlement
- **State Management**: Built-in lifecycle management (pending, paid, expired, canceled)
- **Efficient Indexing**: Creator and recipient indexing for fast queries
- **Real-time Tracking**: Event emission for payment monitoring
- **Security**: Protection against double payments and unauthorized access
- **Anti-spam Protection**: Minimum payment amounts and rate limiting

## System Architecture

### Contract Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    BitPay Tags Contract                     │
├─────────────────────────────────────────────────────────────┤
│  Core Functions          │  Data Storage    │  Utilities    │
│  ├─ create-payment-tag   │  ├─ payment-tags │  ├─ Validation│
│  ├─ fulfill-payment-tag  │  ├─ creator-index│  ├─ Events    │
│  ├─ cancel-payment-tag   │  ├─ recipient-idx│  ├─ Statistics│
│  └─ expire-payment-tag   │  └─ stats        │  └─ Admin     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      sBTC Token Contract                    │
│                    (External Dependency)                    │
└─────────────────────────────────────────────────────────────┘
```

### Data Models

#### Payment Tag Structure

```clarity
{
  creator: principal,           ; Tag creator address
  recipient: principal,         ; Payment recipient address
  amount: uint,                ; Payment amount in sBTC units
  created-at: uint,            ; Creation block height
  expires-at: uint,            ; Expiration block height
  memo: (optional string),     ; Optional payment description
  state: string,               ; Current state (pending/paid/expired/canceled)
  payment-tx: (optional buff), ; Transaction hash when fulfilled
  payment-block: (optional uint) ; Block height when fulfilled
}
```

#### Index Structures

- **Creator Index**: Maps creators to their tag IDs for efficient querying
- **Recipient Index**: Maps recipients to relevant tag IDs
- **Statistics**: Tracks contract usage metrics

## Data Flow

### 1. Payment Tag Creation

```
User Input → Validation → State Storage → Index Updates → Event Emission
    ↓           ↓            ↓              ↓              ↓
[Amount,    [Min Amount,  [payment-tags  [creator-index, [payment-tag-
 Recipient,  Expiration,   map update]    recipient-idx]  created event]
 Expiration, Self-pay,
 Memo]       Auth checks]
```

### 2. Payment Fulfillment

```
Fulfill Request → Tag Validation → sBTC Transfer → State Update → Event
      ↓               ↓              ↓              ↓           ↓
[Tag ID Input] → [Exists, Pending, → [Transfer to → [Mark as → [Fulfillment
                  Not Expired]       Recipient]     PAID]      Event]
```

### 3. State Transitions

```
PENDING ──fulfill──→ PAID
   │
   ├──cancel──→ CANCELED (by creator only)
   │
   └──expire──→ EXPIRED (after expiration block)
```

## Installation & Deployment

### Prerequisites

- Stacks blockchain testnet/mainnet access
- Clarity development environment
- sBTC token contract deployed and accessible

### Deployment Steps

1. **Update Configuration**

   ```clarity
   ;; Update with actual sBTC contract address
   (define-constant SBTC-CONTRACT 'YOUR-SBTC-CONTRACT-ADDRESS)
   ```

2. **Deploy Contract**

   ```bash
   clarinet deploy --testnet
   ```

3. **Verify Deployment**

   ```clarity
   (contract-call? .bitpay-tags get-contract-info)
   ```

## Usage Examples

### Creating a Payment Request

```clarity
;; Create a payment request for 1000000 sBTC units (0.01 sBTC)
;; Expires in 1440 blocks (~10 days)
(contract-call? .bitpay-tags create-payment-tag
  'ST1RECIPIENT-ADDRESS-HERE
  u1000000
  u1440
  (some "Invoice #12345 - Web Development"))
```

### Fulfilling a Payment

```clarity
;; Fulfill payment tag with ID 1
(contract-call? .bitpay-tags fulfill-payment-tag u1)
```

### Querying Payment Status

```clarity
;; Get payment tag details
(contract-call? .bitpay-tags get-payment-tag u1)

;; Get all tags created by a user
(contract-call? .bitpay-tags get-creator-tags 'ST1USER-ADDRESS)

;; Get all tags where user is recipient
(contract-call? .bitpay-tags get-recipient-tags 'ST1USER-ADDRESS)
```

## API Reference

### Public Functions

| Function | Parameters | Description |
|----------|------------|-------------|
| `create-payment-tag` | recipient, amount, expires-in-blocks, memo | Creates a new payment request |
| `fulfill-payment-tag` | tag-id | Fulfills an existing payment request |
| `cancel-payment-tag` | tag-id | Cancels a payment request (creator only) |
| `expire-payment-tag` | tag-id | Marks an expired tag as expired |

### Read-Only Functions

| Function | Parameters | Returns | Description |
|----------|------------|---------|-------------|
| `get-payment-tag` | tag-id | Payment tag data | Retrieves specific tag details |
| `get-creator-tags` | creator | List of tag IDs | Gets tags created by user |
| `get-recipient-tags` | recipient | List of tag IDs | Gets tags for recipient |
| `get-contract-stats` | stat-key | Statistic value | Retrieves usage statistics |
| `can-expire-tag` | tag-id | Boolean | Checks if tag can be expired |

## Security Considerations

### Access Control

- Only tag creators can cancel their own tags
- Only contract deployer can pause/unpause contract
- Self-payments are prevented

### State Management

- Tags can only be fulfilled when in PENDING state
- Expired tags cannot be fulfilled
- Double payment protection through state checks

### Input Validation

- Minimum payment amounts prevent spam
- Maximum expiration limits prevent indefinite tags
- Memo length validation
- Principal validation for recipients

## Configuration

### Constants

```clarity
MIN-PAYMENT-AMOUNT: u1000        ; 0.00001 sBTC minimum
MAX-EXPIRATION-BLOCKS: u4320     ; ~30 days maximum
MAX-TAGS-PER-USER: u100          ; Index size limit
```

### Error Codes

- `ERR-TAG-EXISTS` (100): Tag already exists
- `ERR-NOT-PENDING` (101): Tag not in pending state
- `ERR-INSUFFICIENT-FUNDS` (102): Insufficient sBTC balance
- `ERR-NOT-FOUND` (103): Tag not found
- `ERR-UNAUTHORIZED` (104): Access denied
- `ERR-EXPIRED` (105): Tag has expired
- `ERR-INVALID-AMOUNT` (106): Invalid amount specified
- `ERR-EMPTY-MEMO` (107): Empty memo provided
- `ERR-MAX-EXPIRATION-EXCEEDED` (108): Expiration too long
- `ERR-INVALID-RECIPIENT` (109): Invalid recipient address
- `ERR-SELF-PAYMENT` (110): Cannot pay yourself

## Events

The contract emits events for all major operations:

- `payment-tag-created`: New tag creation
- `payment-tag-fulfilled`: Tag payment completion
- `payment-tag-canceled`: Tag cancellation
- `payment-tag-expired`: Tag expiration
- `contract-pause-toggled`: Contract pause state changes

## Statistics Tracking

- `tags-created`: Total number of tags created
- `tags-fulfilled`: Total number of successful payments
- `tags-canceled`: Total number of canceled tags
- `tags-expired`: Total number of expired tags

## Development

### Testing

```bash
clarinet test
```

### Local Development

```bash
clarinet console
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Add tests for new functionality
4. Ensure all tests pass
5. Submit a pull request

## License

MIT License - see LICENSE file for details

## Support

For issues and questions:

- Create an issue in the GitHub repository
- Review the contract documentation
- Check event logs for debugging

---

**Note**: This contract requires sBTC tokens for operation. Ensure proper sBTC contract integration before deployment to mainnet.
