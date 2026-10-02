# Wallets

This page is for builders of wallets and apps. It lists what a wallet must do to build LUCKY-20 transactions without harming its users. The payloads and layouts are on the operation pages: [Deploy](../operations/deploy.md), [Mine](../operations/mine.md) and [Send](../operations/send.md).

A wallet needs one thing beyond a normal Bitcoin wallet: a view of which of its outputs hold tokens, and how many. That view is computed from Bitcoin blocks by the LUCKY-20 rules. It must have processed every block up to the chain tip before the wallet relies on it.

## Every transaction

- Put the payload in **one** OP_RETURN output, in its exact compact form: no spaces, the keys in the order shown on the operation pages. Build it from the template, byte for byte: a payload that does not parse makes the transaction a plain spend, and in the SEND layout every token on its inputs then goes to output 0, the fee address.
- Make every token output **546 sats**. This is a wallet convention; the rules accept a token output of any value.
- Pay the protocol fee as **one output of the exact amount** to `bc1phk23psaqmq4rlsjeet79xpt65n9v2hvrv97ezc6c4rpld4s2shwqa9qx9n`: 5,460 sats for a DEPLOY, 546 sats for a MINE or a SEND.
- **Always emit the residual output** of a SEND and of a fill (output 2), even when the residual is 0.
- Put BTC change last. It is optional: drop it when it would be below dust. It never carries tokens.
- For P2TR inputs, include `tapInternalKey`. For P2WPKH inputs, a `witnessUtxo` is enough.
- Sign with `SIGHASH_DEFAULT` (P2TR) or `SIGHASH_ALL` (P2WPKH). Use `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` only for a listing: the tokens of an input signed that way move only with a SEND of their ticker that is applied, and otherwise go to the output with the same index as the input (this applies only to a P2TR key-path or P2WPKH signature, and only when that output exists and is not an OP_RETURN output; see the [Glossary](../glossary.md)).
- Signal replace-by-fee, so that a slow transaction can be sped up.

## Never spend tokens as fees

If a token output is used to pay fees, its tokens go wherever that transaction routes its input pool, often to someone else. So, when choosing fee inputs:

1. Leave out every output of **546 sats or less**.
2. Leave out every output that your token view shows as a token output.
3. If your list of outputs is not asset-safe (it may include Ordinals or Runes outputs), also leave out every output of **10,000 sats or less**, and pick the **largest** outputs first.
4. Leave out the inputs of your own unconfirmed transactions until they confirm or leave the mempool. Spending one again would replace the earlier transaction.

## Deploy

1. Check that the ticker is free in your token view, and that the view has processed every block up to the chain tip.
2. Build the reference DEPLOY (see [Deploy](../operations/deploy.md)). Signal replace-by-fee on every input, and sign every input with `SIGHASH_DEFAULT` (P2TR) or `SIGHASH_ALL` (P2WPKH): only such inputs count toward the deployer. Keep a BTC change output large enough to pay for a replacement with a higher fee.
3. Right before signing, check again that the ticker is still free.
4. Broadcast with a fast fee.
5. While it waits, tell the user that the ticker is visible in the mempool and that another DEPLOY of the same ticker that pays more can confirm first. Offer to raise the fee (replace-by-fee).
6. After it confirms, show the result from your token view: registered to this DEPLOY or one of its fee-raised replacements, or to another DEPLOY, in which case this one's fees are spent and not refunded. Show it as provisional until its block has 6 confirmations.

## Mine

- Show the **remaining supply** before building a MINE. After the supply is used up, a MINE still pays the 546-sat fee and is credited 0.
- Output 0 receives the yield. Pay it to the user's own address.
- Offer MINE once the ticker's DEPLOY has confirmed: a MINE in the same block as the DEPLOY is invalid. A one-block reorganization that moves the DEPLOY can still leave a MINE sent at its first confirmation in the DEPLOY's block or before it; that MINE pays its fees and is credited nothing.
- The credit is known only after the MINE confirms. Show it from the last hex digit of the confirming block's hash. It can still change if that block is replaced, so show it as provisional until the block has 6 confirmations.

## Send

- Build the layout on the [Send](../operations/send.md) page.
- Choose inputs that hold at least `AMT` of the ticker. If they hold less, the SEND does not apply and every token goes to the residual output (output 2).

## Trading

### Listing (seller)

| Part | Value |
| --- | --- |
| Inputs | Exactly one: the token output, holding at least 546 sats. It must hold **one ticker only**; the listing sells its whole balance. |
| Outputs | Exactly one: `price_sats` to **the same script** as the input. |
| `price_sats` | At least 546, and at least the token output's own BTC value. |
| Signature | Input 0 signed with `SIGHASH_SINGLE \| SIGHASH_ANYONECANPAY` (`0x83`), not finalized. |
| `nLockTime` | 0 |
| Version | 1 or 2 |
| Input 0 `nSequence` | `0x80000000` or higher |

A listing signed with another version or `nSequence` can never be filled. Any BTC on the token output above `price_sats` goes to the buyer, so sell tokens from a fresh 546-sat token output. To sell part of a balance, or one ticker of a mixed output, first split it with a SEND to yourself.

### Fill (buyer)

| vout | value | to |
| --- | --- | --- |
| 0 | `price_sats` | seller (from the listing, unchanged) |
| 1 | 546 sats | buyer (receives the tokens) |
| 2 | 546 sats | buyer (residual output, always present) |
| 3 | 546 sats | fee address |
| 4 | 0 | OP_RETURN with the SEND payload |
| 5 | change | buyer, BTC change (optional) |

The buyer adds BTC inputs after input 0, signs them, completes input 0 with the seller's signature, and broadcasts. Keep the listing at input 0. If two buyers fill the same listing, only one transaction confirms.

A spend of the listed output counts as a trade only when it carries the seller's `0x83` signature and the matching output pays the seller at least `price_sats`. The tokens reach the buyer only when the transaction is a SEND of the listed ticker that applies — with its 546-sat fee output. Without it, the seller is paid and the tokens stay with the seller: the trade is recorded as the seller's own.

After a spend of a listing without the fee, the tokens sit on the seller's payment output, which holds the price in BTC. Move them to a 546-sat token output with a SEND to yourself before spending that output from any other wallet: a plain BTC payment from it would carry the tokens along.

One applied SEND of a ticker moves the tokens of every listing of that ticker in the transaction, so a transaction that completes several listings of one ticker pays the fee once. Keep the first listing at input 0 and put each further one at input 3 or later, with its seller paid at the output of the same index. A listing at input 1 or 2 would put its seller's payment where the SEND's output 1 or 2 must be.

Before signing, the buyer's wallet checks:

1. The listing has exactly one input and one output, and input 0 carries a signature with sighash type `0x83`.
2. Output 0 pays `price_sats` to the script of input 0.
3. The value that the listing records for input 0 equals the real value of the token output.
4. In your token view, the token output is unspent and holds exactly the listed amount of the ticker, and more than 0.
5. The ticker's market is open: the block that completed its supply has at least 6 confirmations.
6. A second source agrees. Fetch the transaction that created the token output from your own node or a block explorer. Its script and value must match, and its payload must put tokens on that output:
   - a **mined** output is output 0 of a MINE. Its amount must be the tier of its block hash. Only in the block where the ticker reached its supply cap can it be a smaller partial credit;
   - a **sent** output is output 1 of a SEND (then `AMT` must equal the listed amount) or its residual output: output 2, or, when output 2 is missing or is an OP_RETURN output, the first output that is not an OP_RETURN output.

   Any other output, any other operation, or no LUCKY-20 payload is a disagreement. A mined output 0, or a sent output 1 that is not also the residual output, whose creating transaction has no 546-sat fee output is a disagreement too: that MINE or SEND put nothing there. When input 1 of the creating SEND is signed as a listing, output 1 may hold that listing's tokens instead, so the amount is not independently verified.

Stop if the checks disagree. The creating transaction cannot confirm every amount: a partial credit depends on the remaining supply, and the amount on a residual output depends on the inputs. In those cases, tell the user that the amount is not independently verified, and sign only after an extra, explicit confirmation.

Set `AMT` to the listed amount only when the second source confirmed it. Otherwise set it to 1: the SEND then applies whenever the listed output holds any of the ticker, and every token on it reaches the buyer (1 on output 1, the rest on output 2). A fill whose `AMT` is larger than the listed output's real balance does not apply, and the seller keeps both the price and the tokens.

### Cancel

A signed listing can be filled by anyone who holds it. The only real cancel is to **move the tokens on-chain** with a SEND to yourself. Listing the same token output again at a lower price does not cancel the earlier signed listing, and neither does its expiry from the order book: anyone who kept a copy can still fill it.

Sign the cancel with your wallet's ordinary signature type (`SIGHASH_DEFAULT` or `SIGHASH_ALL`), never `0x83`. A spend of the listed output signed `0x83` that pays you the price is a sale.

If a fill with a low fee is waiting in the mempool, the cancel must replace it under the replace-by-fee rules (BIP125). It must pay a higher fee rate than that fill, by at least the incremental relay fee. Its total fee must be at least the fees of that fill and of everything that depends on it, plus the relay fee for its own size.

## The reference app

The app at [luckyprotocolai.com](https://luckyprotocolai.com) works with the UniSat wallet. It holds no keys. It builds unsigned PSBTs and asks UniSat to sign them.

To pay fees, the app starts from the list of plain BTC outputs that UniSat reports for the connected address. UniSat leaves inscription and rune outputs out of that list. The app then checks every listed output against the confirmed UTXO set of its own Bitcoin node, and uses it only when:

- the node has it as unspent, with at least 1 confirmation;
- its script is the connected address's script, and its value is the value UniSat reported;
- the token view shows no tokens on it.

An output that the node does not have as confirmed yet waits for its confirmation. The app also applies the rules in [Never spend tokens as fees](#never-spend-tokens-as-fees): it leaves out outputs of 546 sats or less and the inputs of its own unconfirmed transactions. If the check cannot be made, the app builds nothing and asks the user to try again.
