# Building Batch Transactions: Multi-Recipient Settlements & Complex Flows

## Introduction

Batch transactions enable atomic execution of multiple operations across multiple parties on the Midnight blockchain. This tutorial covers composing multi-recipient transactions, understanding block weight constraints, and implementing reliable settlement patterns.

## Multi-Party Transaction Composition

Midnight supports composing transactions that affect multiple recipients in a single atomic operation:

```typescript
const tx = await midnight.composeTransaction({
    operations: [
        { type: 'transfer', to: recipient1, amount: 100 },
        { type: 'transfer', to: recipient2, amount: 200 },
        { type: 'contractCall', contract: escrowContract, method: 'release' }
    ]
});
```

## Block Weight Constraints

Error 1010 occurs when a transaction exceeds block weight limits. Strategies to handle this:

### Splitting Large Operations

```typescript
async function batchWithSplit(operations, maxPerTx = 5) {
    const batches = [];
    for (let i = 0; i < operations.length; i += maxPerTx) {
        const batch = operations.slice(i, i + maxPerTx);
        const tx = await midnight.composeTransaction({ operations: batch });
        batches.push(await tx.submit());
    }
    return batches;
}
```

### Guaranteed vs Fallible Segments

```typescript
const tx = await midnight.composeTransaction({
    segments: [
        { operations: criticalOps, type: 'guaranteed' },
        { operations: optionalOps, type: 'fallible' }
    ]
});
```

## Best Practices

1. Group related operations into logical batches
2. Monitor block weight usage with `estimateBlockWeight()`
3. Use fallible segments for non-critical operations
4. Implement retry logic with exponential backoff

## Conclusion

Batch transactions enable efficient multi-party settlements on Midnight. Understanding block weight constraints and segment types ensures reliable execution of complex flows.
