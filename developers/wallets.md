# Wallets

This page is for builders of wallets and apps. It lists what a wallet must do to build LUCKY-20 transactions without harming its users. The payloads and layouts are on the operation pages: [Deploy](../operations/deploy.md), [Mine](../operations/mine.md) and [Send](../operations/send.md). The complete rules are in the [specification](https://luckyprotocolai.com/PROTOCOL.md).

A wallet needs one thing beyond a normal Bitcoin wallet: a view of which of its outputs hold tokens, and how many. That view is computed from Bitcoin blocks by the rules of the specification. It must have processed every block up to the chain tip before the wallet relies on it.

## Every transaction

- Put the payload in **one** OP_RETURN output.
- Make every token output **546 sats**. This is a wallet convention; the rules accept a token output of any value.
- Pay the protocol fee as **one output of the exact amount** to `bc1phk23psaqmq4rlsjeet79xpt65n9v2hvrv97ezc6c4rpld4s2shwqa9qx9n`: 5,460 sats for a reveal, 546 sats for a MINE or a SEND, nothing for a COMMIT.
- **Always emit the residual output** of a SEND (output 3) and of a fill (output 4), even when the residual is 0.
- Put BTC change last. It is optional: drop it when it would be below dust. It never carries tokens.
- For P2TR inputs, include `tapInternalKey`. For P2WPKH inputs, a `witnessUtxo` is enough.
- Signal replace-by-fee, so that a slow transaction can be sped up.

## Never spend tokens as fees

If a token output is used to pay fees, its tokens go wherever that transaction routes its input pool, often to someone else. So, when choosing fee inputs:

1. Leave out every output of **546 sats or less**.
2. Leave out every output that your token view shows as a token output.
3. If your list of outputs is not asset-safe (it may include Ordinals or Runes outputs), also leave out every output of **10,000 sats or less**, and pick the **largest** outputs first.
4. Leave out the inputs of your own unconfirmed transactions until they confirm or leave the mempool. Spending one again would replace the earlier transaction.

## Deploy

1. Check that the ticker is free in your token view.
2. Make a salt: 16 random bytes, written as 32 lower-case hex characters.
3. Build the payload `P = LUCKY-20|DEPLOY|<TICKER>|<SALT>` and take the script `S` of your own address, which the commit output will pay. Compute `H = SHA-256( P ‖ S )`. See [How to compute `H`](../operations/deploy.md#how-to-compute-h).
4. **Save the ticker, the salt and the script before signing.** Without the salt, the COMMIT can never be revealed.
5. Broadcast the COMMIT. Wait for **two** confirmations. Never broadcast the REVEAL before that: a REVEAL sent at the first confirmation can, after a one-block reorganization, confirm in the COMMIT's own block. There it does not count, the COMMIT is used up and the ticker is public.
6. Right before the reveal, check that the COMMIT is confirmed and valid, that its commit output pays your address and is unspent, that its `H` matches, and that the ticker is still free. If the COMMIT is invalid, start again with a new salt.
7. Broadcast the REVEAL with a fast fee, spending the commit output as **input 0**, within 2,016 blocks of the COMMIT, and not when fewer than 6 blocks of that window remain.

Do not spend the commit output in any other way: any spend uses up the COMMIT. While a COMMIT is open, and after it expires until its last reveal block has 6 confirmations, sign an output that came from it only with `SIGHASH_ALL` or `SIGHASH_DEFAULT`, never with `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY`.

## Mine

- Show the **remaining supply** before building a MINE. After the supply is used up, a MINE still pays the 546-sat fee and is credited 0.
- Output 0 receives the yield. Pay it to the user's own address.
- Offer MINE only after the ticker's reveal has **2** confirmations: a MINE in the same block as the reveal is invalid, and after a one-block reorganization a MINE sent at the first confirmation can confirm in the reveal's block or before it, and still pay its fees.
- The credit is known only after the MINE confirms. Show it from the last hex digit of the confirming block's hash. It can still change if that block is replaced, so show it as provisional until the block has 6 confirmations.

## Send

- Build the layout on the [Send](../operations/send.md) page.
- Choose inputs that hold at least `AMT` of the ticker. If they hold less, the SEND does not apply and every token goes to the residual output.

## Trading

### Listing (seller)

| Part | Value |
| --- | --- |
| Inputs | Exactly one: the token output. It must hold **one ticker only**; the listing sells its whole balance. |
| Outputs | Exactly one: `price_sats` to **the same script** as the input. |
| `price_sats` | At least 546, and at least the token output's own BTC value. |
| Signature | Input 0 signed with `SIGHASH_SINGLE \| SIGHASH_ANYONECANPAY` (`0x83`), not finalized. |
| `nLockTime` | 0 |
| Version | 1 or 2 |
| Input 0 `nSequence` | `0x80000000` or higher |

A listing signed with another version or `nSequence` can never be filled. Any BTC on the token output above `price_sats` goes to the buyer, so sell tokens from a fresh 546-sat token output. To sell part of a balance, or one ticker of a mixed output, first split it with a SEND to yourself. Never list the commit output of an open COMMIT.

### Fill (buyer)

| vout | value | to |
| --- | --- | --- |
| 0 | `price_sats` | seller (from the listing, unchanged) |
| 1 | 546 sats | buyer (`TO_OUT`, receives the tokens) |
| 2 | 546 sats | fee address |
| 3 | 0 | OP_RETURN `LUCKY-20\|SEND\|<TICKER>\|<AMT>\|1\|4` |
| 4 | 546 sats | buyer (`CHANGE_OUT`, residual output, always present) |
| 5 | change | buyer, BTC change (optional) |

The buyer adds BTC inputs after input 0, signs them, completes input 0 with the seller's signature, and broadcasts. If two buyers fill the same listing, only one transaction confirms.

Before signing, the buyer's wallet checks:

1. The listing has exactly one input and one output, and input 0 carries a signature with sighash type `0x83`.
2. Output 0 pays `price_sats` to the script of input 0.
3. The value that the listing records for input 0 equals the real value of the token output.
4. In your token view, the token output is unspent and holds exactly the listed amount of the ticker, and more than 0.
5. The ticker's market is open: the block that completed its supply has at least 6 confirmations.
6. A second source agrees. Fetch the transaction that created the token output from your own node or a block explorer. Its script and value must match, and its payload must put tokens on that output:
   - a **mined** output is output 0 of a MINE. Its amount must be the tier of its block hash. Only in the block where the ticker reached its supply cap can it be a smaller partial credit;
   - a **sent** output is the `TO_OUT` of a SEND (then `AMT` must equal the listed amount) or its `CHANGE_OUT`.

   Any other output, any other operation, or no LUCKY-20 payload is a disagreement.

Stop if the checks disagree. The creating transaction cannot confirm every amount: a partial credit depends on the remaining supply, and the amount on a residual output depends on the inputs. In those cases, tell the user that the amount is not independently verified, and sign only after an extra, explicit confirmation.

### Cancel

A signed listing can be filled by anyone who holds it. The only real cancel is to **move the tokens on-chain** with a SEND to yourself. Listing the same token output again at a lower price does not cancel the earlier signed listing.

If a fill with a low fee is waiting in the mempool, the cancel must replace it under the replace-by-fee rules (BIP125). It must pay a higher fee rate than that fill, by at least the incremental relay fee. Its total fee must be at least the fees of that fill and of everything that depends on it, plus the relay fee for its own size.

## The reference app

The app at [luckyprotocolai.com](https://luckyprotocolai.com) holds no keys. It builds unsigned PSBTs and asks the connected wallet (UniSat or OKX Wallet) to sign them.
