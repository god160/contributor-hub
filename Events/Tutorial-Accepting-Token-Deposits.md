# Accepting Token Deposits into a Smart Contract: ReceiveShielded & Escrow Patterns on Midnight

## Introduction

The Midnight blockchain provides a unique privacy-preserving architecture where tokens can exist in two distinct forms: **shielded** (private) and **transparent**. When building smart contracts that accept token deposits, understanding how to receive shielded tokens is essential for maintaining user privacy while enabling complex financial logic.

This tutorial explores the core APIs for handling shielded token deposits, including `receiveShielded`, `writeCoin`, `sendShielded`, and `mergeCoinImmediate`. We'll build a practical escrow contract example that demonstrates these patterns in action.

---

## Contract-Held vs User-Held Coins

Before diving into the APIs, it's crucial to understand the fundamental difference between how coins are held:

### User-Held Coins

User-held coins are controlled directly by external accounts. The owner can:
- Send coins to other users or contracts
- Merge multiple smaller coins into larger denominations
- Split larger coins into smaller denominations
- Control when and how coins are spent

### Contract-Held Coins

When a contract receives deposits, the **contract itself becomes the coin holder**. This creates important distinctions:

| Aspect | User-Held | Contract-Held |
|--------|-----------|---------------|
| Control | External key | Contract logic |
| Visibility | Encrypted/ shielded | Transparent to contract |
| Spending | User signature | Contract function calls |
| Tracking | Hidden from observers | Visible internally |

The key insight: contracts can **see** the values of shielded coins they hold, even though external observers cannot. This enables contracts to enforce rules while preserving privacy from the outside world.

---

## QualifiedShieldedCoinInfo Structure

The `QualifiedShieldedCoinInfo` struct represents a validated coin within the Midnight privacy system:

```typescript
struct QualifiedShieldedCoinInfo {
    Field coin_id;           // Unique coin identifier
    Field value;             // Token amount (encrypted)
    Field token_id;          // Asset type identifier
    Field restrict;          // Spending restrictions (if any)
    Field ciphertext: [8];   // Encrypted coin data
}
```

When a coin is **qualified**, it has been verified by the network as valid and unspent. Contracts receive arrays of `QualifiedShieldedCoinInfo` when users invoke deposit functions.

---

## The receiveShielded Function

The `receiveShielded` function is the primary mechanism for contracts to accept private token deposits:

```typescript
import {
    receiveShielded,
    QualifiedShieldedCoinInfo,
    Field
} from '@midnight-ntwrk/midnight-js-types';

interface DepositResult {
    coins: QualifiedShieldedCoinInfo[];
    total_value: bigint;
}

// The contract's public deposit endpoint
async function depositCoins(
    coins: QualifiedShieldedCoinInfo[],
    expected_token_id: Field
): Promise<DepositResult> {
    // Validate input
    if (coins.length === 0) {
        throw new Error('No coins provided for deposit');
    }

    // The receiveShielded call:
    // - Transfers coins from user to contract
    // - Validates all coins are properly signed
    // - Verifies token_id matches expectations
    const received = await receiveShielded(coins, expected_token_id);

    // Calculate total deposited value
    const total_value = coins.reduce((sum, coin) => {
        return sum + extractValue(coin);
    }, 0n);

    return { coins: received, total_value };
}
```

### How receiveShielded Works Internally

1. **Signature Verification**: Confirms the user authorized the transfer
2. **Token Validation**: Ensures the coin's `token_id` matches expectations
3. **Double-Spend Check**: Verifies coins haven't been spent elsewhere
4. **Ownership Transfer**: Updates the coin's controller to the contract address

---

## Writing Coins with writeCoin

The `writeCoin` function creates new coins, which contracts need when **returning** funds to users:

```typescript
import { writeCoin, Field, PrivateInput } from '@midnight-ntwrk/midnight-js-types';

interface CoinCreation {
    coin: QualifiedShieldedCoinInfo;
    ciphertext: [Field, Field, Field, Field, Field, Field, Field, Field];
}

async function createWithdrawalCoin(
    value: bigint,
    token_id: Field,
    recipient_public_key: Field,
    contract_nonce: Field
): Promise<CoinCreation> {
    // Generate the coin's unique identifier
    const coin_id = await deriveCoinId(
        recipient_public_key,
        contract_nonce,
        value
    );

    // Create the coin structure
    const coin = await writeCoin({
        value: value,
        token_id: token_id,
        restrict: Field(0),  // No additional restrictions
        coin_id: coin_id
    });

    return { coin, ciphertext: coin.ciphertext };
}
```

### Practical Example: Withdrawal Function

```typescript
async function withdrawToUser(
    recipient: Field,           // User's public key
    amount: bigint,
    token_id: Field
): Promise<void> {
    // Create a new coin for the recipient
    const { coin } = await createWithdrawalCoin(
        amount,
        token_id,
        recipient,
        this.nonce++
    );

    // Send the coin to the user
    await sendShielded(recipient, [coin]);
}
```

---

## Sending Shielded Tokens with sendShielded

The `sendShielded` function transfers coins from the contract to a recipient:

```typescript
import { sendShielded, Field } from '@midnight-ntwrk/midnight-js-types';

async function sendToRecipient(
    recipient: Field,
    coins: QualifiedShieldedCoinInfo[],
    remaining_coins: QualifiedShieldedCoinInfo[]
): Promise<void> {
    // sendShielded requires:
    // 1. The coins to send
    // 2. The coins to keep (returned as change)
    await sendShielded(recipient, coins, remaining_coins);
}
```

### Important: Coin Management

Unlike traditional transactions, shielded transfers require explicit **coin management**:
- If you send 75 tokens from a 100-token coin, you receive 25 tokens back as change
- The contract must track both spent and remaining coins

---

## Merging Coins with mergeCoinImmediate

When a contract accumulates many small coins, gas costs increase. `mergeCoinImmediate` consolidates multiple coins into fewer, larger denominations:

```typescript
import { 
    mergeCoinImmediate, 
    QualifiedShieldedCoinInfo,
    Field 
} from '@midnight-ntwrk/midnight-js-types';

interface MergeResult {
    merged_coin: QualifiedShieldedCoinInfo;
    change_coins: QualifiedShieldedCoinInfo[];
}

async function consolidateCoins(
    input_coins: QualifiedShieldedCoinInfo[],
    target_count: number  // How many coins to produce
): Promise<MergeResult> {
    if (input_coins.length < 2) {
        throw new Error('Need at least 2 coins to merge');
    }

    // Merge coins to reduce count while preserving total value
    const { coin, change } = await mergeCoinImmediate(
        input_coins,
        target_count,
        extractTokenId(input_coins[0])
    );

    return {
        merged_coin: coin,
        change_coins: change
    };
}
```

### When to Merge

| Scenario | Recommendation |
|----------|----------------|
| Many small deposits | Merge after 10+ coins |
| Gas optimization needed | Merge before large operations |
| Maintaining privacy | Avoid merging too aggressively |

---

## Complete Escrow Contract Example

Here's a complete implementation of a privacy-preserving escrow contract:

```typescript
import { 
    Contract, 
    Field, 
    PrivateInput,
    receiveShielded,
    sendShielded,
    writeCoin,
    mergeCoinImmediate,
    QualifiedShieldedCoinInfo
} from '@midnight-ntwrk/midnight-js-contracts';

// Type definitions for contract state
interface EscrowState {
    deposited_coins: QualifiedShieldedCoinInfo[];
    depositor: Field | null;
    beneficiary: Field;
    release_condition_hash: Field;
    total_deposited: bigint;
    nonce: Field;
}

class ShieldedEscrowContract implements Contract<EscrowState> {
    private state: EscrowState;

    constructor(
        public readonly beneficiary: Field,
        public readonly release_condition_hash: Field
    ) {
        this.state = {
            deposited_coins: [],
            depositor: null,
            beneficiary: beneficiary,
            release_condition_hash: release_condition_hash,
            total_deposited: 0n,
            nonce: Field(0)
        };
    }

    // Deposit tokens into escrow (called by depositor)
    async deposit(
        coins: QualifiedShieldedCoinInfo[],
        depositor_public_key: Field
    ): Promise<void> {
        // Cannot deposit if already funded
        if (this.state.depositor !== null) {
            throw new Error('Escrow already funded');
        }

        // Receive the shielded coins
        const received = await receiveShielded(
            coins,
            extractTokenId(coins[0])
        );

        // Update state
        this.state.depositor = depositor_public_key;
        this.state.deposited_coins = received;
        this.state.total_deposited = this.calculateTotal(received);

        console.log(`Deposited ${this.state.total_deposited} tokens`);
    }

    // Release escrow to beneficiary (requires valid condition proof)
    async release(
        condition_proof: PrivateInput,
        expected_hash: Field
    ): Promise<void> {
        // Verify release conditions
        if (this.state.depositor === null) {
            throw new Error('No deposits to release');
        }

        // Verify the condition hash matches
        const proof_hash = await this.hashProof(condition_proof);
        if (proof_hash !== this.state.release_condition_hash) {
            throw new Error('Release conditions not met');
        }

        // Send all deposited coins to beneficiary
        const all_coins = this.state.deposited_coins;
        
        await sendShielded(
            this.state.beneficiary,
            all_coins,
            []  // No change coins - send everything
        );

        // Clear state
        this.state.depositor = null;
        this.state.deposited_coins = [];
        this.state.total_deposited = 0n;

        console.log('Escrow released to beneficiary');
    }

    // Cancel escrow and return funds to depositor
    async cancel(): Promise<void> {
        if (this.state.depositor === null) {
            throw new Error('No deposits to cancel');
        }

        // Optimized sending - merge if many coins
        let coins_to_send = this.state.deposited_coins;
        
        if (coins_to_send.length > 5) {
            // Consolidate before sending to save gas
            const consolidated = await this.mergeCoins(coins_to_send);
            coins_to_send = consolidated;
        }

        // Return to original depositor
        await sendShielded(
            this.state.depositor,
            coins_to_send,
            []
        );

        // Clear state
        this.state.depositor = null;
        this.state.deposited_coins = [];
        this.state.total_deposited = 0n;

        console.log('Escrow cancelled, funds returned');
    }

    // Helper: Merge coins for gas efficiency
    private async mergeCoins(
        coins: QualifiedShieldedCoinInfo[]
    ): Promise<QualifiedShieldedCoinInfo[]> {
        // Merge down to 3 coins for efficiency
        const { merged_coin, change_coins } = await mergeCoinImmediate(
            coins,
            3,
            extractTokenId(coins[0])
        );

        return [merged_coin, ...change_coins];
    }

    // Helper: Calculate total value
    private calculateTotal(coins: QualifiedShieldedCoinInfo[]): bigint {
        return coins.reduce((sum, coin) => sum + extractValue(coin), 0n);
    }

    // Helper: Hash the release proof
    private async hashProof(proof: PrivateInput): Promise<Field> {
        // Implementation depends on condition logic
        return Field.random(); // Placeholder
    }

    // Read-only: Check escrow status
    getStatus(): { 
        funded: boolean; 
        total: bigint; 
        coin_count: number 
    } {
        return {
            funded: this.state.depositor !== null,
            total: this.state.total_deposited,
            coin_count: this.state.deposited_coins.length
        };
    }
}
```

---

## Security Considerations

When implementing deposit patterns, consider these security aspects:

### 1. Token Validation
```typescript
// ALWAYS verify token_id on deposits
async function safeDeposit(coins: QualifiedShieldedCoinInfo[]): Promise<void> {
    const expected_token = this.allowed_token_id;
    
    for (const coin of coins) {
        if (!tokenIdsMatch(coin.token_id, expected_token)) {
            throw new Error('Invalid token type in deposit');
        }
    }
    
    await receiveShielded(coins, expected_token);
}
```

### 2. Value Overflow Protection
```typescript
function safeAdd(total: bigint, addition: bigint, max: bigint): bigint {
    if (total + addition > max) {
        throw new Error('Deposit would exceed maximum');
    }
    return total + addition;
}
```

### 3. Reentrancy Prevention
- Always update internal state **before** external calls
- Use checks-effects-interactions pattern

---

## Best Practices Summary

| Pattern | Recommendation |
|---------|----------------|
| **Deposits** | Always validate token_id before `receiveShielded` |
| **Coin Tracking** | Maintain internal mapping of coin_id to coin data |
| **Gas Optimization** | Merge coins when count exceeds 5-10 |
| **Privacy** | Don't log coin values publicly |
| **Error Handling** | Provide clear error messages for failed deposits |
| **Partial Withdrawals** | Track spent coins separately from remaining coins |

---

## Conclusion

Accepting shielded token deposits in Midnight contracts requires understanding the lifecycle of private coins within the contract context. The key functions—`receiveShielded`, `writeCoin`, `sendShielded`, and `mergeCoinImmediate`—form the foundation of privacy-preserving contract logic.

By following the patterns and security considerations in this tutorial, you can build sophisticated financial applications that maintain user privacy while enforcing trustless contract logic. The escrow contract example demonstrates how these primitives combine to create practical, production-ready solutions.

Remember: contracts can see the values of shielded coins they hold, but external observers cannot. This unique property enables powerful combination of privacy and programmable finance.