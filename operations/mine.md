# Mine

A MINE creates new tokens of a registered ticker. It names the ticker and nothing else. The hash of the Bitcoin block that confirms it sets the amount.

## Example

```
LUCKY-20|MINE|LUCKY
```

Fields are separated by `|` and read by position.

| Key | Required? | Description |
| --- | --- | --- |
| `protocol` | Yes | Field 0. Always `LUCKY-20`. |
| `op` | Yes | Field 1. `MINE`, in upper case. |
| `tick` | Yes | Field 2. A registered ticker: 1 to 8 characters, `A`–`Z` and `0`–`9`. |

There is no amount field and no output index. Nothing in the payload changes the yield. The new tokens always go to output 0.

## Yield

The last hex digit of the confirming block's hash, as block explorers print it, sets the tier:

| Last hex digit | Tier (tokens) | Digits out of 16 |
| --- | --- | --- |
| `f` | 1,000 | 1 |
| `c` `d` `e` | 500 | 3 |
| `7` `8` `9` `a` `b` | 200 | 5 |
| `0` `1` `2` `3` `4` `5` `6` | 100 | 7 |

Expected yield: (1 × 1,000 + 3 × 500 + 5 × 200 + 7 × 100) / 16 = **262.5 tokens** per MINE.

The credit is the smaller of the yield and the remaining supply.

Example: a MINE of `LUCKY` confirms in a block whose hash ends in `…a8c3`. The last digit is `3`, so output 0 receives 100 `LUCKY`.

## Reference transaction

| vout | value | to |
| --- | --- | --- |
| 0 | 546 sats | you: the **yield output** |
| 1 | 546 sats | fee address `bc1phk23psaqmq4rlsjeet79xpt65n9v2hvrv97ezc6c4rpld4s2shwqa9qx9n` |
| 2 | 0 | OP_RETURN `LUCKY-20\|MINE\|<TICKER>` |
| 3 | change | you (optional) |

## Notes

- **Nobody chooses the amount.** The block hash is found after the MINE is already in the block. Nobody can choose it or know it in advance.
- **Use the hash as explorers print it.** Take the block hash exactly as a block explorer or Bitcoin Core prints it. It starts with zeros.
- **Same block, same tier.** Every MINE in one block receives the same tier, because they share one block hash. MINEs are credited in block order, so near the cap the earlier MINEs of a block are credited first. Only the end of the supply can credit less than the tier.
- **A MINE is valid when** the ticker was registered in an earlier block, output 0 exists and is not an OP_RETURN output, and an output pays exactly 546 sats to the fee address. An invalid MINE credits nothing. A MINE in the same block as the ticker's reveal is invalid, whoever sends it, the deployer included: minting starts in the block after the reveal. A wallet therefore waits for the reveal's second confirmation before offering MINE.
- **Final at 6 confirmations.** Until its block has 6 confirmations, the credit is provisional. If Bitcoin replaces that block (a reorganization), the MINE is credited again from the hash of the block that confirms it in the new chain, and its tier can change.
- **Open mint.** Anyone can mine any registered ticker, as often as they like. There is no per-address cap.
- **After the supply is used up**, a MINE is credited 0 but still pays the 546-sat fee. Check the remaining supply before you mine.
- **Tokens on the inputs** of a MINE, of every ticker, also go to output 0, even when the MINE is invalid. They are burned only when output 0 is missing or is an OP_RETURN output.
- **Never pay for a MINE with a token output.** See [Wallets](../developers/wallets.md#never-spend-tokens-as-fees).
- Transactions confirmed below block 969,300 do nothing.
- Complete rules: the [specification](https://luckyprotocolai.com/PROTOCOL.md).
