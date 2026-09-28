# Send

A SEND moves tokens of one ticker from the token outputs a transaction spends to one of its outputs. Everything else goes to a second output that you name. A market fill is also a SEND.

## Example

```
LUCKY-20|SEND|LUCKY|1200|0|3
```

Fields are separated by `|` and read by position.

| Key | Required? | Description |
| --- | --- | --- |
| `protocol` | Yes | Field 0. Always `LUCKY-20`. |
| `op` | Yes | Field 1. `SEND`, in upper case. |
| `tick` | Yes | Field 2. The ticker to move: 1 to 8 characters, `A`–`Z` and `0`–`9`. |
| `amt` | Yes | Field 3. `AMT`: whole tokens to move, 1 to 21,000,000. Decimal digits only: no sign, no decimal point, no leading zero. |
| `to` | Yes | Field 4. `TO_OUT`: the index of the output that receives `AMT`, 0 to 255. |
| `change` | Yes | Field 5. `CHANGE_OUT`: the index of the output that receives everything else, 0 to 255. It must differ from `TO_OUT`. |

All six fields are required. A payload with a field missing, an extra field, or a field that breaks its rule is not a SEND. The transaction is then treated like one without a LUCKY-20 payload (see Default routing below).

This example moves 1,200 `LUCKY` to output 0 and everything else to output 3.

## Reference transaction

| vout | value | to |
| --- | --- | --- |
| 0 | 546 sats | recipient (`TO_OUT`, receives `AMT`) |
| 1 | 546 sats | fee address `bc1phk23psaqmq4rlsjeet79xpt65n9v2hvrv97ezc6c4rpld4s2shwqa9qx9n` |
| 2 | 0 | OP_RETURN `LUCKY-20\|SEND\|<TICKER>\|<AMT>\|0\|3` |
| 3 | 546 sats | you (`CHANGE_OUT`, the **residual output**, always present) |
| 4 | change | you, BTC change (optional; it never carries tokens) |

## Result

The SEND is **applied** when all of these hold:

- the inputs hold at least `AMT` of the ticker;
- output `TO_OUT` exists and is not an OP_RETURN output;
- an output pays exactly 546 sats to the fee address.

Then `AMT` goes to output `TO_OUT`. The rest of that ticker, and every other ticker on the inputs, goes to output `CHANGE_OUT`.

When the SEND is not applied, nothing goes to output `TO_OUT`. All tokens on the inputs go to output `CHANGE_OUT`.

```
inputs     { LUCKY: 1,500, ABC: 500 }
payload    LUCKY-20|SEND|LUCKY|1200|0|3
vout0      { LUCKY: 1,200 }             the recipient
vout3      { LUCKY: 300, ABC: 500 }     back to you
```

## Notes

- **One ticker per SEND.** Only the named ticker moves to `TO_OUT`. One SEND can therefore split a token output that holds several tickers.
- **Keep the residual output.** Output 3 exists even when it receives nothing. Never drop it as dust. It keeps tokens off the BTC change output, so a later plain BTC spend of the change cannot carry them away.
- **An unusable `CHANGE_OUT`.** If output `CHANGE_OUT` is missing or is an OP_RETURN output, what it would receive goes to the default output instead: the first output that is not an OP_RETURN output.
- **Default routing.** A Bitcoin transaction without a LUCKY-20 payload moves all tokens on its inputs to its default output. The tokens are not destroyed. The rare cases in which tokens are burned are listed in the specification.
- **Tokens never go to an OP_RETURN output.**
- **A trade is a SEND.** A fill uses `TO_OUT` = 1 and `CHANGE_OUT` = 4. See [Wallets](../developers/wallets.md#trading).
- Transactions confirmed below block 969,300 do nothing.
- Complete rules: the [specification](https://app.luckyprotocolai.com/PROTOCOL.md).
