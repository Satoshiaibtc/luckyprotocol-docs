# Deploy

A deploy registers a new ticker with a fixed supply of 21,000,000 tokens. It takes two transactions: a **COMMIT**, then a **REVEAL** (the `DEPLOY` payload) in a later block. The COMMIT shows only a hash, so nobody can copy the ticker from the mempool. The deployer receives no tokens. Every token comes from a [MINE](mine.md).

Fields are separated by `|` and read by position. The tables list them in order.

## Step 1: COMMIT

### Example

```
LUCKY-20|COMMIT|1ac55b4c608ed7c39eb3dbcecaf04c41222d5b3c37b6343477c9a91d4a6f33fc
```

| Key | Required? | Description |
| --- | --- | --- |
| `protocol` | Yes | Field 0. Always `LUCKY-20`. |
| `op` | Yes | Field 1. `COMMIT`, in upper case. |
| `hash` | Yes | Field 2. `H`: exactly 64 lower-case hex characters. See "How to compute `H`" below. |

The COMMIT payload is always 80 bytes.

### How to compute `H`

```
H = SHA-256( P ‖ S )

P = the UTF-8 bytes of the exact REVEAL payload you will send in step 2
S = the raw output script (scriptPubKey) of this COMMIT's output 0,
    without the length byte that comes before it in a serialized transaction
‖ = plain byte concatenation: the bytes of P, then the bytes of S
```

Example, a test vector from the specification:

| | Value |
| --- | --- |
| `P` | `LUCKY-20\|DEPLOY\|LUCKY\|000102030405060708090a0b0c0d0e0f` |
| `S` (P2TR, 34 bytes) | `51200102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20` |
| `H` | `1ac55b4c608ed7c39eb3dbcecaf04c41222d5b3c37b6343477c9a91d4a6f33fc` |

The same `P` under another script gives another `H`. A real salt must be random.

### Reference transaction

| vout | value | to |
| --- | --- | --- |
| 0 | 546 sats | your own address: the **commit output** (its script is `S`) |
| 1 | 0 | OP_RETURN `LUCKY-20\|COMMIT\|<H>` |
| 2 | change | you (optional) |

A COMMIT pays **no protocol fee**.

## Step 2: REVEAL

### Example

```
LUCKY-20|DEPLOY|LUCKY|000102030405060708090a0b0c0d0e0f
```

| Key | Required? | Description |
| --- | --- | --- |
| `protocol` | Yes | Field 0. Always `LUCKY-20`. |
| `op` | Yes | Field 1. `DEPLOY`, in upper case. |
| `tick` | Yes | Field 2. The ticker: 1 to 8 characters, `A`–`Z` and `0`–`9`. |
| `salt` | Yes | Field 3. 16 random bytes, written as exactly 32 lower-case hex characters. The same salt that was used to compute `H`. |

There is no supply field and no amount field. The supply is always 21,000,000, and the block hash sets the amount of each MINE.

### Reference transaction

| Position | value | to |
| --- | --- | --- |
| input 0 | 546 sats | the commit output from step 1 (it must be input 0) |
| inputs 1+ | | your BTC, to pay the fees |
| vout 0 | 546 sats | you (proof output) |
| vout 1 | 5,460 sats | fee address `bc1phk23psaqmq4rlsjeet79xpt65n9v2hvrv97ezc6c4rpld4s2shwqa9qx9n` |
| vout 2 | 0 | OP_RETURN `LUCKY-20\|DEPLOY\|<TICKER>\|<SALT>` |
| vout 3 | change | you (optional) |

### The REVEAL registers the ticker when all of these hold

1. Its input 0 spends the commit output of a COMMIT that is not invalid (see the first note below).
2. That COMMIT's `H` equals SHA-256 of this payload and the commit output's script.
3. The COMMIT confirmed at or after block 969,300.
4. The COMMIT confirmed in an earlier block: the REVEAL is at least 1 block after it.
5. The REVEAL confirms at most 2,016 blocks after the COMMIT.
6. An output pays exactly 5,460 sats to the fee address.
7. The ticker is not registered yet.

## Notes

- **The commit output must be a normal output.** It must exist, must not be an OP_RETURN output, and must pay a standard address. Otherwise the COMMIT is invalid and can never be revealed.
- **Block order decides.** The first valid REVEAL of a ticker, by block height and then position in the block, registers it. The age of the COMMIT gives no priority.
- **A commit is used once.** Any transaction that spends the commit output uses up the COMMIT, whether it is a REVEAL or not. A COMMIT whose commit output is still unspent 2,016 blocks later (about two weeks) expires.
- **A copy is useless.** `H` covers the committer's own script. A copy of `H` in another COMMIT sits on another script, so no payload can match it. A copy of a REVEAL does not spend your commit output.
- **The deployer is the committer**: the address that the commit output pays.
- **Keep the salt.** Save the ticker, the salt and the script before you sign the COMMIT. Without the salt, the COMMIT can never be revealed.
- **Protect an open commit.** While a COMMIT is open (not yet revealed or expired), never sign its commit output with `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` and never list it for sale. This also applies to a commit output that someone else's COMMIT paid to your address. Such a signature covers only one input and one output, so its holder could complete it into a REVEAL that names you as the deployer of a ticker you never chose.
- **Every deploy needs a COMMIT.** A DEPLOY without a salt (`LUCKY-20|DEPLOY|<TICKER>`) never registers a ticker.
- **Tokens on the inputs** of a COMMIT or a REVEAL go to the default output: the first output that is not an OP_RETURN output. In the reference layouts, that is the commit output of a COMMIT and the proof output of a REVEAL.
- Wallet steps: [Wallets](../developers/wallets.md#deploy). Complete rules: the [specification](https://app.luckyprotocolai.com/PROTOCOL.md).
