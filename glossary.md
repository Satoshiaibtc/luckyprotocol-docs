# Glossary

| Term | Meaning |
| --- | --- |
| **Activation height** | Block 969,300. Transactions in earlier blocks do nothing. |
| **Bitcoin miner** | The party that finds a Bitcoin block. Not the same as a minter. |
| **Block hash** | The hash of a Bitcoin block, written the way Bitcoin nodes and block explorers print it (it starts with zeros). Its last hex digit sets the tier of every MINE in that block. |
| **Burn** | Tokens that are routed to no output. This happens only when the rules leave them no usable output. Burned tokens do not lower the minted amount. |
| **Carrier** | The specification's name for a token output. |
| **COMMIT** | `LUCKY-20\|COMMIT\|<H>`. Step 1 of a deploy. It shows only a hash of the future reveal. No protocol fee. |
| **Commit output** | Output 0 of a COMMIT, paid to the committer's own address. The REVEAL must spend it as input 0. Its script is part of `H`. The specification calls it the commit carrier. |
| **Committer** | The address that the commit output pays. It becomes the deployer when its REVEAL registers the ticker. |
| **Credit** | The tokens a MINE actually receives: the smaller of its yield and the remaining supply. |
| **Default output** | The first output of a transaction that is not an OP_RETURN output. |
| **Default routing** | A transaction that spends token outputs without a LUCKY-20 payload moves all their tokens to its default output. |
| **DEPLOY** | The payload of a REVEAL: `LUCKY-20\|DEPLOY\|<TICKER>\|<SALT>`. |
| **Deployer** | The committer of the COMMIT whose commit output the registering REVEAL spent. The deployer receives no tokens. |
| **Expected yield** | The average credit of a MINE while supply remains: (1 × 1,000 + 3 × 500 + 5 × 200 + 7 × 100) / 16 = 262.5 tokens. |
| **Fee address** | `bc1phk23psaqmq4rlsjeet79xpt65n9v2hvrv97ezc6c4rpld4s2shwqa9qx9n`. It receives every protocol fee. |
| **Fill** | A buyer's transaction that completes a listing into a SEND. It pays the seller and gives the tokens to the buyer in one step. |
| **H** | The commit hash: SHA-256 of the exact reveal payload followed by the commit output's script. 64 lower-case hex characters. |
| **Input pool** | All tokens on the outputs a transaction spends, counted per ticker. |
| **Listing** | A one-input, one-output transaction signed by a seller with `SIGHASH_SINGLE \| SIGHASH_ANYONECANPAY`. It sells the whole balance of one ticker on one token output. |
| **MINE** | `LUCKY-20\|MINE\|<TICKER>`. Creates new tokens on output 0. The block hash sets the amount. |
| **Minted amount** | The total credited to all MINEs of a ticker. It never decreases. |
| **Minted out** | A ticker whose minted amount has reached 21,000,000. Also called fully minted. It stays minted out, and its market opens. |
| **Minter** | The sender of a MINE. |
| **OP_RETURN output** | An output that holds data and can never be spent. It never holds tokens. |
| **Payload** | The text in the OP_RETURN output, such as `LUCKY-20\|MINE\|LUCKY`. At most 80 bytes. |
| **Protocol fee** | An output of an exact amount to the fee address: 5,460 sats for a reveal, 546 sats for a MINE or a SEND, none for a COMMIT. |
| **Residual output** | The 546-sat output that receives what a SEND does not move: output 3 of a SEND, output 4 of a fill. Always present. |
| **REVEAL** | Step 2 of a deploy. Its payload is `LUCKY-20\|DEPLOY\|<TICKER>\|<SALT>`. It registers the ticker. Protocol fee 5,460 sats. |
| **Salt** | 16 random bytes (32 lower-case hex characters) in the reveal payload. It keeps the ticker secret until the reveal. |
| **SEND** | `LUCKY-20\|SEND\|<TICKER>\|<AMT>\|<TO_OUT>\|<CHANGE_OUT>`. Moves `AMT` of one ticker to `TO_OUT` and everything else to `CHANGE_OUT`. |
| **Specification** | The complete rulebook, with test vectors: [PROTOCOL.md](https://app.luckyprotocolai.com/PROTOCOL.md). |
| **Supply** | 21,000,000 tokens for every ticker. Fixed by the rules. Not a field of any payload. |
| **Ticker** | 1 to 8 characters, `A`–`Z` and `0`–`9`. The first valid reveal registers it. |
| **Tier** | One of the four yields: 100, 200, 500 or 1,000 tokens. |
| **Token output** | A Bitcoin output that holds LUCKY-20 tokens. By wallet convention it holds 546 sats; the rules accept any value. |
| **Unit price** | The listing price in sats divided by the number of tokens: sats per whole token. |
| **Yield** | The tier a MINE receives from the last hex digit of its block hash: `f` → 1,000; `c`–`e` → 500; `7`–`b` → 200; `0`–`6` → 100. |
