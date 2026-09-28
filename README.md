# LUCKY-20: Fair-Launch Tokens Minted by the Bitcoin Block Hash

LuckyProtocol · [app.luckyprotocolai.com](https://app.luckyprotocolai.com) · Activation at Bitcoin block 969,300

**Abstract.** A token on Bitcoin should not let any person decide who receives how much. In most token standards, people set the amounts: a deployer picks the supply and may keep a share, and each minter picks how much to claim. LUCKY-20 lets Bitcoin decide. A MINE transaction names a ticker and nothing else. The last hex digit of the hash of the block that confirms it sets the credit: 100, 200, 500 or 1,000 tokens, 262.5 on average. The minter, the deployer and the project cannot choose that hash, and anyone can check it from public block data. Every ticker has a fixed supply of 21,000,000 tokens, with no premine, no allocation and no per-address cap. Each operation is one small OP_RETURN payload in an ordinary Bitcoin transaction, and tokens sit on Bitcoin outputs: there is no inscription, no sidechain and no bridge. A new ticker is registered in two steps, commit and reveal. The commitment is bound to the committer's own output script, so a registration seen in the mempool cannot be copied. When a ticker is fully minted, its tokens trade through seller-signed listings that settle atomically on-chain, with no custodian.

## At a glance

- **Minted by Bitcoin itself.** The last hex digit of the confirming block's hash sets each MINE's yield: `f` → 1,000; `c`–`e` → 500; `7`–`b` → 200; `0`–`6` → 100. The expected yield is 262.5 tokens per MINE. Nobody chooses it: not the minter, not the deployer, not the project.
- **Fair launch.** 21,000,000 tokens per ticker. Open mint. No premine, no allocation, no per-address cap. The deployer receives no tokens.
- **Pure Bitcoin L1.** One OP_RETURN payload of at most 80 bytes per transaction. No inscriptions, no sidechain, no bridge. Tokens live on Bitcoin outputs and move with ordinary Bitcoin transactions.
- **Deploys that cannot be copied.** Commit, then reveal. The commitment covers the committer's own output script, so a copy of it is useless.
- **Non-custodial market.** A seller signs a listing. A buyer completes it into one on-chain transaction that pays the seller and moves the tokens together. A ticker's market opens only when it is fully minted.
- **Fixed, public fees.** 546 sats per MINE or SEND and 5,460 sats per deploy, each paid as an exact output to one published address.
- **Open to verification.** Every balance follows from Bitcoin blocks, starting at block 969,300. Anyone can recompute it.

## 1. Introduction

Tokens on Bitcoin are usually issued by people. A deployer picks the supply and the amount per mint, and sometimes keeps a share. Each minter picks how much to claim, up to a limit. Whoever sets these numbers can favor someone, and every holder must trust that they did not. Some standards also need large inscriptions, or a second chain and a bridge.

What is needed is a token whose issuance is set by Bitcoin rather than by a person. Every mint amount should come from data that no participant controls and that anyone can check. The supply should be fixed and open to everyone from the start. The tokens should live in ordinary Bitcoin transactions. Trading them should not require handing them to anyone.

This document describes such a standard. The amount of each mint is read from the hash of the Bitcoin block that confirms it. A ticker goes through four steps:

1. **Commit.** A COMMIT hides the new ticker behind a hash.
2. **Reveal.** 1 to 2,016 blocks later, a REVEAL names the ticker and registers it with a supply of 21,000,000.
3. **Mine.** Anyone sends MINEs. The hash of the confirming block sets the credit of each MINE, until the supply is used up.
4. **Trade.** When the ticker is fully minted, its market opens. Holders list tokens, and buyers fill the listings on-chain.

Section 3 explains the minting rule. The other sections build the token system around it.

## 2. Tokens on Bitcoin Outputs

Every LUCKY-20 operation is an ordinary Bitcoin transaction with one OP_RETURN output. That output holds a short text payload of at most 80 bytes, with fields separated by `|`. Field 0 is always `LUCKY-20`. Field 1 names the operation. There are four operations and no others:

| Operation | Payload | Purpose |
| --- | --- | --- |
| COMMIT | `LUCKY-20\|COMMIT\|<H>` | Step 1 of a deploy. Hides the ticker behind a hash. |
| DEPLOY | `LUCKY-20\|DEPLOY\|<TICKER>\|<SALT>` | Step 2 of a deploy, the reveal. Registers the ticker. |
| MINE | `LUCKY-20\|MINE\|<TICKER>` | Creates new tokens on output 0 of the transaction. |
| SEND | `LUCKY-20\|SEND\|<TICKER>\|<AMT>\|<TO_OUT>\|<CHANGE_OUT>` | Moves tokens to chosen outputs. |

Tokens are bound to transaction outputs. An output that holds tokens is a *token output*. The specification calls it a *carrier*. Whoever can spend it owns its tokens. One output can hold several tickers:

| Output | Value | Tokens |
| --- | --- | --- |
| `txid_a:0` | 546 sats | LUCKY 1,000 |
| `txid_b:3` | 546 sats | LUCKY 200 and ABC 500 |

There are no accounts, no contract and no second ledger. By wallet convention a token output holds 546 sats. Amounts are whole tokens, with no decimals.

When a transaction spends token outputs, their tokens form one *input pool*, counted per ticker. The payload says which outputs receive the pool. A transaction that spends token outputs but has no LUCKY-20 payload moves the whole pool to its first output that is not an OP_RETURN output. This is *default routing*. An ordinary wallet that spends a token output therefore moves the tokens. It does not destroy them. Tokens never go to an OP_RETURN output. They are burned only in the few cases where the rules leave them no usable output. The specification lists these cases.

## 3. Minting by Block Hash

A MINE names a ticker and nothing else:

```
LUCKY-20|MINE|<TICKER>
```

It states no amount. When the transaction confirms, its yield is read from the hash of the block that contains it, written the way Bitcoin nodes and block explorers print it. Only the last hex digit counts:

| Last hex digit of the block hash | Tier (tokens) | Digits out of 16 | Probability |
| --- | --- | --- | --- |
| `f` | 1,000 | 1 | 6.25% |
| `c` `d` `e` | 500 | 3 | 18.75% |
| `7` `8` `9` `a` `b` | 200 | 5 | 31.25% |
| `0` `1` `2` `3` `4` `5` `6` | 100 | 7 | 43.75% |

The last digit of a block hash is evenly spread. Each of the 16 digits is equally likely, and the digit of one block does not depend on any other block. The expected yield of one MINE is:

```
E = (1 × 1,000 + 3 × 500 + 5 × 200 + 7 × 100) / 16
  = 4,200 / 16
  = 262.5 tokens
```

The yield is credited to output 0 of the MINE. Here is the rule applied to the hash of Bitcoin block 0:

```
  MINE tx      LUCKY-20|MINE|LUCKY
     |
     |  confirmed in a Bitcoin block
     v
  block hash   000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f
                                                                              |
     +------------------------------------------------------------------------+
     |  last hex digit = f
     v
  tier         f -> 1,000 | c-e -> 500 | 7-b -> 200 | 0-6 -> 100
     |
     v
  credit       1,000 tokens to output 0 of the MINE
```

**Who decides the amount.** Nobody chooses it.

- The minter only names a ticker, and every MINE pays the same fee. The minter signs and broadcasts the MINE before the block exists. The hash is found later, with the transaction already inside the block.
- The deployer sets no amounts. The supply and the tiers are the same for every ticker.
- The project sets nothing. The rule is a fixed function of public data.
- A Bitcoin miner cannot pick the hash either, because proof-of-work hashes cannot be predicted. A Bitcoin miner could discard a block it found and search again. It would give up that block's subsidy and fees, and the next hash would again be unknown.

All MINEs confirmed in the same block share one block hash, so they all receive the same tier. The yield is the outcome of the block hash and nothing else.

## 4. Supply and Fair Launch

Every ticker has a supply of exactly 21,000,000 tokens. The supply is fixed by the rules. It is not a field of any payload, and a deployer cannot set another number. There is no premine and no allocation: the deployer receives no tokens. Every token of every ticker comes from a MINE.

Minting is open. Anyone can mine any registered ticker, from any address, as often as they like. There is no per-address cap. This also means that a large minter can mint a large part of a supply. The remaining supply is public at every block, so every participant can see how close the cap is.

Each MINE is credited the smaller of its yield and the remaining supply. MINEs are credited in block order, so near the cap the earlier MINEs of a block are credited first. Every tier and the supply are multiples of 100, so the supply ends at exactly 21,000,000. A MINE confirmed after that is credited 0 but still pays its protocol fee. A wallet therefore shows the remaining supply before it builds a MINE.

A ticker is *minted out*, or fully minted, when its credited total reaches 21,000,000. The credited total never decreases. Tokens burned later do not reopen minting, so a minted-out ticker stays minted out.

## 5. Deploying a Ticker

A ticker is 1 to 8 characters, `A`–`Z` and `0`–`9`. The first valid registration of a ticker is final.

A deploy that names its ticker in plain text could be copied from the mempool. Someone could confirm the copy first by paying a higher fee. So a ticker is registered in two transactions:

1. **COMMIT**: `LUCKY-20|COMMIT|<H>`. Output 0 is the *commit output*, paid to the committer's own address. `H` is a hash of the future reveal payload `P` and of the commit output's script `S`:

   ```
   H = SHA-256( P ‖ S )
   ```

   `P` holds the ticker and a salt of 16 random bytes, so nobody can learn the ticker from `H`. The COMMIT pays no protocol fee.
2. **REVEAL**: `LUCKY-20|DEPLOY|<TICKER>|<SALT>`. It registers the ticker when its input 0 spends the commit output, `H` matches, the REVEAL confirms 1 to 2,016 blocks after the COMMIT, it pays the 5,460-sat protocol fee, and the ticker is still free.

```
 block N       COMMIT   vout0: commit output (script S, pays the committer)
                        OP_RETURN: LUCKY-20|COMMIT|H      H = SHA-256(P ‖ S)
                           |
                           |  1 to 2,016 blocks later
                           v
 block N+k     REVEAL   input 0: spends the commit output
                        OP_RETURN: P = LUCKY-20|DEPLOY|TICKER|SALT
                        5,460 sats to the fee address
```

Why a copy fails:

- A copy of `H` in someone else's COMMIT sits on another output script. To reveal it, the copier would need a payload whose hash with the copier's own script equals `H`. Finding one is not feasible. The copier does not even know the ticker or the salt.
- A copy of a REVEAL taken from the mempool cannot spend the committer's commit output. Only the committer can sign for it.
- A copier who commits the revealed payload under a new script must first wait for that COMMIT to confirm. By then the original REVEAL, sent with a fast fee, has normally confirmed, and the ticker is taken.

The deployer is the committer: the address that the commit output pays. When two valid REVEALs name the same ticker, the first in block order registers it. The age of the COMMIT gives no priority.

## 6. Moving Tokens

```
LUCKY-20|SEND|<TICKER>|<AMT>|<TO_OUT>|<CHANGE_OUT>
```

A SEND moves `AMT` whole tokens of one ticker to output `TO_OUT`. Everything else in the input pool goes to output `CHANGE_OUT`: the rest of that ticker and all of every other ticker. The two indices must differ. Amounts are whole numbers from 1 to 21,000,000.

```
inputs     { LUCKY: 1,500, ABC: 500 }
payload    LUCKY-20|SEND|LUCKY|1200|0|3
vout0      { LUCKY: 1,200 }             the recipient
vout3      { LUCKY: 300, ABC: 500 }     back to the sender
```

A SEND applies only when the pool holds at least `AMT`, output `TO_OUT` exists and is not an OP_RETURN output, and the fee output is present. Otherwise nothing moves to `TO_OUT`, and the whole pool goes to `CHANGE_OUT`. If output `CHANGE_OUT` is missing or is an OP_RETURN output, what it would receive goes to the transaction's first output that is not an OP_RETURN output.

Because routing is per ticker, one SEND can split a token output that holds several tickers. And because of default routing (Section 2), tokens also move with any Bitcoin transaction that spends them, even one built by a wallet that knows nothing about LUCKY-20.

## 7. Trading

Bitcoin has no contract that can hold tokens during a sale. A LUCKY-20 trade is therefore one Bitcoin transaction built by two people:

- **Listing.** The seller signs a transaction with one input, the token output, and one output, the price paid to the seller's own script. The signature type is `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY`. It covers only that input and that output, so the listing is valid only inside a transaction that pays the seller in full.
- **Fill.** A buyer adds BTC inputs and the outputs that make the transaction a SEND of the tokens to the buyer, then signs and broadcasts it. In one transaction the seller is paid and the buyer receives the tokens. If it does not confirm, nothing moves.
- **Cancel.** A signed listing stays valid for anyone who holds it, so an off-chain cancel means nothing. The seller cancels by moving the tokens on-chain with a SEND back to the seller.

```
 listing (signed by the seller)         fill (completed by the buyer)
 in0   token output, e.g. 1,200 tokens  in0   the seller's token output
 out0  price -> seller                  in1+  the buyer's BTC
                                        out0  price -> seller
                                        out1  546 sats -> buyer (the tokens)
                                        out2  546 sats -> fee address
                                        out3  OP_RETURN  LUCKY-20|SEND|<TICKER>|1200|1|4
                                        out4  546 sats -> buyer (residual)
                                        out5  BTC change -> buyer
```

A listing sells the whole balance of one ticker on one token output. To sell part of a balance, the seller first splits it with a SEND back to the seller. If two buyers fill the same listing, only one transaction can confirm, and the other buyer spends nothing.

**The market opens when a ticker is minted out.** Before that, the order book accepts no listing of the ticker. While anyone can still mint the ticker for the fee, a listing would put a price on something anyone can mint. It would also let a deployer sell into a distribution that is not finished.

Nobody holds funds or tokens for anyone: not the app, not the order book. There is no bonding curve, no pooled liquidity and no market maker. Prices are what sellers ask and buyers pay. The order book can hide or delay a listing, but it cannot move a seller's coins or tokens. Before signing, the buyer's wallet checks the listed token output against the chain (Section 8).

## 8. Verification

The state of every ticker is a function of Bitcoin blocks and the published rules. No trusted party reports it. Anyone who has the block data can recompute it:

1. Start at block 969,300. Ignore every earlier block.
2. Take each transaction in block order. Read its payload, apply the rule for its operation, and route its input pool.
3. For each MINE, take the last hex digit of the confirming block's hash and look up the tier.

Anyone who applies the rules to the same blocks gets the same balances. The yield of a MINE always comes from the block that confirms it in the chain that Bitcoin keeps.

A single yield can be checked by hand. The first blocks of Bitcoin show how the rule reads a hash. They are far below block 969,300 and hold no MINE:

| Height | Block hash | Last digit | Tier |
| --- | --- | --- | --- |
| 0 | `000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f` | `f` | 1,000 |
| 1 | `00000000839a8e6886ab5951d76f411475428afc90947ee320161bbf18eb6048` | `8` | 200 |
| 2 | `000000006a625f06636b8bb6ac7b960a8d03705d1ace08b1a19da3fdcc99ddbd` | `d` | 500 |
| 4 | `000000004ebadb55ee9096c9a2f8880e09da59c0d68b1c228da88e48844a1485` | `5` | 100 |

A buyer checks a listed token output the same way. The buyer fetches the transaction that created it from a Bitcoin node or a block explorer, compares its script and value, and reads its payload. For an output created by a MINE, the buyer recomputes the yield from the block hash.

## 9. Fees

| Operation | Protocol fee |
| --- | --- |
| COMMIT | none |
| DEPLOY (reveal) | 5,460 sats |
| MINE | 546 sats |
| SEND (a fill is a SEND) | 546 sats |

A protocol fee is one output of exactly that amount to the published fee address:

```
bc1phk23psaqmq4rlsjeet79xpt65n9v2hvrv97ezc6c4rpld4s2shwqa9qx9n
```

An output of any other amount does not count. The fees are fixed by the rules. They do not depend on amounts or prices. Without its fee output, an operation does not take effect: a MINE credits nothing, a SEND moves nothing to `TO_OUT`, and a reveal registers nothing. A MINE pays its fee even when no supply remains.

The Bitcoin network fee is separate, as in any Bitcoin transaction. A reference MINE has two 546-sat outputs: the token output, which the minter keeps, and the protocol fee. The network fee is paid in addition.

## 10. Calculations

**Expected yield.** From Section 3, one MINE is expected to credit 262.5 tokens.

**MINEs to mint out.** At the expected yield, a ticker is minted out after

```
21,000,000 / 262.5 = 80,000 MINEs
```

**Bounds.** While supply remains, every MINE is credited between 100 and 1,000 tokens. So minting out takes at least 21,000 MINEs (every one in the top tier) and at most 210,000 MINEs (every one in the lowest tier):

```
21,000,000 / 1,000 =  21,000
21,000,000 /   100 = 210,000
```

**Blocks, not MINEs.** All MINEs in one block share one tier. The average of 262.5 therefore holds over many blocks, not over many MINEs in one block. A ticker minted in a few blocks can end far from 80,000 MINEs. A ticker minted over many blocks tends to end close to it.

## 11. Conclusion

We have described a token standard on Bitcoin in which no person sets the mint amounts. Each MINE is credited 100, 200, 500 or 1,000 tokens, 262.5 on average, by the outcome of the hash of the block that confirms it. The minter, the deployer and the project cannot choose that value, and anyone can check it. Each ticker has a fixed supply of 21,000,000, minted openly with no premine, no allocation and no per-address cap. Operations are small OP_RETURN payloads, and tokens sit on Bitcoin outputs. Ticker registration uses a commit and a reveal bound to the committer's own script, so it cannot be copied. Tokens of a minted-out ticker trade through seller-signed listings that settle in one on-chain transaction. The whole state can be recomputed from Bitcoin blocks alone.

---

| Resource | Where |
| --- | --- |
| Operations | [Deploy](operations/deploy.md) · [Mine](operations/mine.md) · [Send](operations/send.md) |
| Building transactions | [Wallets](developers/wallets.md) |
| Specification: the complete rules, with test vectors | [app.luckyprotocolai.com/PROTOCOL.md](https://app.luckyprotocolai.com/PROTOCOL.md) |
| App | [app.luckyprotocolai.com](https://app.luckyprotocolai.com) |

Where this document and the specification differ, the specification is correct.
