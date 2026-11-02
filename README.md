# pepper-curve-jupiter

Jupiter's `Amm` interface for **Pepper's USDC bonding curve**, the `pepper_curve` program on Solana
mainnet. Pepper (https://pepper.fan) launches one token per YouTube video or channel on a bonding curve
priced in USDC; this crate lets Jupiter quote and route swaps on those curves.

| | |
|---|---|
| Program | `CDPwGWGWbAE8FL5f2oVdzfkTutrnsKJcAcgqWktC3dwZ` (mainnet and devnet) |
| Deployed build | executable hash `7d0b4fdee2c0f0f2b23c0c7783c69606e863ea34bab6341061a50487f10b679a`, verified by OtterSec (2026-11-01) |
| Verified source | [`Pepperoni-team/custom-program` at `1256ece`](https://github.com/Pepperoni-team/custom-program/tree/1256ece6865e23ec5c35069bfbe5f4d2c7335d77), the commit OtterSec's build matched ([status](https://verify.osec.io/status/CDPwGWGWbAE8FL5f2oVdzfkTutrnsKJcAcgqWktC3dwZ)) |
| Upgrade authority | a Squads v4 multisig vault |
| Global USDC config | `CuC2WeLhhwVA65gYCKk2Zu12kC5uGJSyriFKWJwkT9F9` (`UsdcConfig`, one address on every cluster) |
| Market key | the curve's PDA (`CurveUsdc`, seeds and bump checked) |
| Label | `Pepper` |
| Interface | `jupiter-amm-interface` at `70f56e0` (GitHub main of 2026-10-22, 0.7.0) |

## Layout

The paths are those of Pepper's own repository, so nothing is rewritten for this copy:

- `tests-onchain/pepper-curve-jupiter/`: the crate (`src/`), its parity suite (`tests/`) and fixtures, its
  own workspace and `Cargo.lock`.
- `programs/pepper_curve_math/`: the curve's arithmetic, the very functions the program's USDC handlers
  call, with no Solana dependency, and its golden vectors (`vectors/curve-v1.json`). The quote is the
  program's own math, never a copy.
- `lib/solana/pepper-curve/idl/pepper_curve.json`: the program's IDL, which the unit tests read to hold
  the account layouts and the instructions' account lists to it.

## The market

- **Quote**: exact-in only, on the curve's counters (`CurveUsdc`); no vault balance is read. A buy's
  `in_amount` is what the curve charges, below the amount asked on the buy that fills the curve at
  graduation (a partial fill); `fee_amount` is in USDC: the 1% platform fee and, on a revenue curve
  inside its raise, the creator's and Pepper's raise shares.
- **`is_active`**: bonding phase, neither pause, and past the launch window (the first 5 minutes, when
  the program caps a buy at 1% of the supply to the buyer's own token account).
- **Swap**: the program's own `buy_exact_in` (13 accounts) or `sell_exact_in` (11 accounts) over their
  fixed lists; account 8, the creator's USDC account, is passed exactly while a revenue curve's raise is
  open. After graduation a token trades on its Raydium CPMM pool, which Jupiter routes already.

## The `Swap` variant

Until Jupiter gives the venue a `Swap` variant of its own, the SDK travels as
`Swap::Placeholder { data: Some([direction]) }`: one byte, `0` a buy and `1` a sell
(`DIRECTION_BUY`, `DIRECTION_SELL` in `src/swap.rs`), which `swap_instruction_data` turns, with the
amounts, into the program's instruction data. Nothing is needed on Jupiter's side for it now. Once
Jupiter assigns Pepper a variant of its own, we move the SDK to it: a small change in `src/swap.rs`,
with the quote, the market and the account lists unchanged.

## Tests

```bash
cd tests-onchain/pepper-curve-jupiter
cargo test --locked
```

41 tests, all passing on the deployed program's bytes: 26 unit tests; 7 pools written from the golden
vectors; 4 devnet curves (an equity curve fresh and one buy away from graduation, a revenue curve inside
and past its raise); 4 mainnet curves (two revenue curves inside their raise, two equity curves),
snapshotted by Jupiter's test kit into `tests/fixtures/accounts/` and committed, so the suite runs
offline. Jupiter's kit (`assert_pool_parity`) checks each quote's `out_amount` against what the program
delivers in LiteSVM; the suite's own harness checks the source's change against `in_amount`, the fee
accounts against `fee_amount`, both vaults and the curve's counters.

`tests/fixtures/pepper.so` is the program as deployed on mainnet (`solana program dump`, the hash
above). New snapshots: delete a pool's folder, then run its test file with `RPC=<a mainnet or devnet
endpoint>`.

## License

MIT.
