---
name: crypto-subscribe
description: Guide users through subscribing to Caves using cryptocurrency payments (Ethereum or Solana). Handles the complete workflow - from showing pricing/benefits, creating payment sessions, to verifying transactions.
---

# Crypto Subscription Skill

## Overview

This skill guides users through subscribing to Caves using cryptocurrency payments. It orchestrates a 3-step workflow using the `caves__crypto_subscribe` MCP tool:

1. **Get Configuration** - Display current pricing, supported chains, and subscription benefits
2. **Create Payment Session** - Generate payment address and expected amount
3. **Submit Transaction** - Verify payment and activate subscription

## When to Invoke This Skill

Use this skill when the user wants to:
- Subscribe to Caves using cryptocurrency
- Pay for Caves subscription with Ethereum or Solana
- See crypto subscription pricing and options
- Check what chains and tokens are supported
- Subscribe without using a credit card

**Keywords that should trigger this skill:**
= "subscribe to caves"
- "subscribe with crypto"
- "pay with ethereum/solana"
- "crypto subscription"
- "subscribe using USDC/ETH/SOL"
- "how to subscribe with cryptocurrency"

## Prerequisites

Before starting, verify:

1. **Caves Account**: User must have an account at itscaves.com
2. **MCP Access**: Verify with `caves__my_shadows()` - should return user's perspectives
3. **Crypto Wallet**: User should have MetaMask (Ethereum) or Phantom (Solana)
4. **Sufficient Balance**: User needs crypto to cover subscription + gas/transaction fees

If MCP tools are not accessible, inform the user they need to set up Caves MCP integration first.

## Workflow Instructions for Claude

### Step 1: Display Configuration

**Objective**: Show user current pricing, supported chains, and subscription benefits.

**Instructions**:
1. Call `caves__crypto_subscribe({ step: 'get_config' })`
2. Parse the response to extract:
   - Pricing tiers (1, 3, 6, 12 months) with discounts
   - Supported chains (ethereum, solana) and tokens (ETH, USDC, SOL)
   - Subscription benefits (fire bonuses, berf status, monthly fire)
   - Session expiration (24 hours)

3. Present this information clearly to the user:

**Example**:
```
Claude: "Here are the current crypto subscription options for Caves:

📊 Pricing Tiers:
• 1 month: $10 (no discount)
• 3 months: $27 ($30 base, 10% discount)
• 6 months: $48 ($60 base, 20% discount)
• 12 months: $90 ($120 base, 25% discount)

⛓️ Supported Chains & Tokens:
• Ethereum: ETH, USDC
• Solana: SOL, USDC

🎁 Subscription Benefits:
• One-time fire bonus: 30,000 fire
• Monthly fire: 30,000 fire per lunar cycle
• Berf status (subscriber badge)

⏱️ Payment Sessions:
• You'll have 24 hours to complete payment after creating a session

Which subscription length would you like? (1, 3, 6, or 12 months)
And which chain/token would you prefer? (e.g., Ethereum USDC, Solana SOL)"
```

### Step 2: Create Payment Session

**Objective**: Generate a payment address and expected amount for the user.

**Instructions**:
1. Confirm user's choices:
   - Months (1, 3, 6, or 12)
   - Chain (ethereum or solana)
   - Token (ETH, USDC, or SOL)

2. Call `caves__crypto_subscribe({ step: 'create_session', months: X, chain: 'Y', token: 'Z' })`

3. Parse the response to extract:
   - `paymentId` (save this for step 3)
   - `paymentAddress` (where user sends crypto)
   - `expectedAmount` (exact amount to send)
   - `priceUsd` (USD value)
   - `expiresAt` (session expiration time)

4. Display clear payment instructions:

**Example**:
```
Claude: "✅ Payment session created! Here's how to complete your subscription:

💰 Payment Details:
• Amount: 27.5 USDC (for 3 months)
• Send to: 0xabc123...def789
• Chain: Ethereum
• ⚠️ Send EXACTLY 27.5 USDC or slightly more
• ⚠️ Do not send less than this amount

⏰ Session expires in 24 hours: Feb 27, 2026 at 2:30 PM

📝 Steps:
1. Open MetaMask and select Ethereum network
2. Send EXACTLY 27.5 USDC to: 0xabc123...def789
3. Wait for transaction confirmation
4. Copy your transaction hash
5. Come back and provide the transaction hash

💡 Tips:
• For ETH/SOL payments, send 5% extra to account for price fluctuations
• For stablecoins (USDC), send the exact amount
• Double-check the address before sending
• Make sure you're on the correct network

Once you've sent the transaction, provide me with the transaction hash."
```

### Step 3: Submit Transaction Hash

**Objective**: Verify the payment and activate the subscription.

**Instructions**:
1. Wait for user to provide transaction hash
2. Call `caves__crypto_subscribe({ step: 'submit_transaction', paymentId: 'X', txHash: 'Y' })`
   - Use the `paymentId` from Step 2
   - Use the `txHash` provided by the user

3. Handle the response:

**Success Response**:
```
Claude: "🎉 Payment verified! Your subscription is now active!

✅ Subscription Details:
• Status: Active
• Period ends: May 27, 2026
• Prepaid months: 3

🎁 Subscription Benefits Activated:
• 30,000 fire bonus awarded (one-time)
• 30,000 fire monthly (every lunar cycle)
• Berf status enabled

Your transaction: 0x123abc...
Amount paid: 27.5 USDC

Welcome to Caves Premium! 🔥"
```

**Error Responses** (handle these gracefully):

- **Insufficient amount**:
  ```
  "⚠️ Transaction verification failed: Insufficient amount received.

  Expected: 27.5 USDC
  Received: 25.0 USDC

  Please create a new payment session and send the correct amount."
  ```

- **Wrong recipient**:
  ```
  "⚠️ Transaction verification failed: Payment sent to wrong address.

  Please create a new payment session and carefully verify the payment address."
  ```

- **Not confirmed**:
  ```
  "⚠️ Transaction not yet confirmed on blockchain.

  Please wait a few minutes for confirmation, then try submitting again."
  ```

- **Transaction hash already used**:
  ```
  "⚠️ This transaction hash was already used for a previous subscription.

  Each transaction can only be used once."
  ```

- **Session expired**:
  ```
  "⚠️ Payment session expired (24 hours passed).

  Please create a new payment session to subscribe."
  ```

- **Active Stripe subscription**:
  ```
  "⚠️ You have an active Stripe subscription.

  Please cancel your Stripe subscription first before subscribing with crypto."
  ```

### Optional Step 4: Check Payment Status

**Objective**: Allow users to check the status of their payment session.

**Instructions**:
1. If user wants to check status, call:
   `caves__crypto_subscribe({ step: 'check_status', paymentId: 'X' })`

2. Display the status:

**Example**:
```
Claude: "Payment Session Status:

📊 Details:
• Status: pending
• Months: 3
• Price: $27 USD
• Chain: ethereum
• Token: USDC
• Expected amount: 27.5 USDC
• Payment address: 0xabc...def
• Expires at: Feb 27, 2026 at 2:30 PM

Status: Waiting for transaction..."
```

## Troubleshooting Guide

Provide this information if user encounters issues:

### Common Issues

1. **"Transaction verification failed"**
   - **Insufficient amount**: For ETH/SOL, send 5% more to account for price fluctuations
   - **Wrong recipient**: Double-check payment address matches session
   - **Not confirmed**: Wait for blockchain confirmation (a few minutes)

2. **"Payment session expired"**
   - Sessions expire after 24 hours
   - Create a new session to try again

3. **"Transaction hash already used"**
   - Each transaction can only be used once
   - Create new session if you need to subscribe again

4. **"You have an active Stripe subscription"**
   - Cannot have both Stripe and crypto subscriptions
   - Cancel Stripe subscription first at itscaves.com/account

5. **Transaction stuck/pending**
   - Check transaction on Etherscan (Ethereum) or Solscan (Solana)
   - May need to wait longer for confirmation
   - Gas price may have been too low (Ethereum)

### Security Best Practices

Remind users to:
- ✅ Verify payment address is correct before sending
- ✅ Double-check amount and network
- ✅ Use reputable wallets (MetaMask, Phantom)
- ❌ Never share private keys
- ❌ Only send to addresses from API response
- ✅ Check transaction on blockchain explorer before submitting hash

## Example Full Conversation

```
User: "I want to subscribe with crypto"

Claude: [Calls get_config, displays pricing and options]
       "Which subscription length would you like?"

User: "3 months with Ethereum USDC"

Claude: [Calls create_session]
       "✅ Payment session created!
       Send 27.5 USDC to: 0xabc...def
       [Full payment instructions]"

User: "I sent it, here's the hash: 0x123..."

Claude: [Calls submit_transaction]
       "🎉 Payment verified! Your subscription is now active!
       [Subscription details and benefits]"
```

## Additional Notes

- **Fire balance**: Users should have fire balance to create connections, but subscriptions award 30,000 fire immediately
- **Session management**: Each payment session is single-use and expires after 24 hours
- **Price fluctuations**: For volatile assets (ETH, SOL), we recommend sending 5% extra
- **Stablecoins**: For USDC, send exact amount (no need for extra)
- **Blockchain confirmations**: Ethereum takes ~1-5 minutes, Solana takes ~30 seconds
- **Transaction fees**: Users pay blockchain gas/transaction fees in addition to subscription price

## Error Handling

If `caves__crypto_subscribe` returns an error:
1. Parse the error message
2. Provide helpful context based on error type
3. Suggest next steps (create new session, wait longer, check blockchain explorer)
4. Offer to help troubleshoot further

Never leave users stuck - always provide actionable next steps.
