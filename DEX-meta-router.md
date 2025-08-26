
# Any Trades Swapp Aggrigator

A **DEX meta-router** for any ERC-20↔ERC-20 swap on Base. It searches **all configured V2 routers** *and* **all configured V3 quoters** (Uniswap/Pancake style), including **cross-DEX, multi-hop paths**, then executes the single best route through an on-chain aggregator contract. It does **not** require the entire path to exist on one exchange.

# How it finds the best route

For a user’s `(from, to, amountIn)`:

1. **Path search space**

   * Direct: `from → to`
   * One-hop via curated “mid” tokens (e.g., WETH/USDC/etc.) from `chains.json`: `from → mid → to`

2. **Protocol combos**

   * **V2** on each router: `getAmountsOut(amountIn, path[])`
   * **V3** on each quoter: `quoteExactInput(encodedPath, amountIn)` with **all fee-tier combinations**
   * **Cross-protocol composites**

     * **V2→V3**: `from→mid` on a V2 router, then `mid→to` on a V3 router
     * **V3→V2**: `from→mid` on a V3 quoter, then `mid→to` on a V2 router
     * **V2→V2 across different routers**: `from→mid` on Router A, then `mid→to` on Router B

3. **Read-only quoting (fast & safe)**

   * Everything is **static calls** (no transactions), so it’s safe, quick, and doesn’t need gas.
   * We run all quote candidates **in parallel**, decode the results, **dedupe identical routes**, and keep the **highest `amountOut`**.

4. **Best route → executable steps**

   * The frontend converts the chosen quote into **Aggregator contract steps**:

     * **V2 step**: protocol=V2, `router`, `abi.encode(address[])` path
     * **V3 step**: protocol=V3, `router`, `encodedPath` (packed with fees)
     * Composites produce 2 steps (e.g., V2 then V3).
   * It applies user slippage to compute `amountOutMin`.

# Why it works even when one DEX doesn’t have both legs

Because we allow **cross-DEX, cross-protocol chaining** via a mid token. If the best `from→WETH` liquidity is on Pancake V2 and the best `WETH→USDC` is on BaseSwap V3, we stitch them together and still give one clean swap with the best outcome.

# Prices & USD display (separate from execution)

* The **/quote** endpoint is used **only** to get executable quotes.
* For “mid” USD prices (to show `$ amounts` and price impact), the UI calls **`/api/:chainId/addresses?addresses=…`** **once when tokens change**.

  * The prices API handles **token decimals correctly** (even if a token isn’t in your list, it reads on-chain), computes **token→WETH** and **WETH→USDC** using the same V2/V3 quoter logic, and returns **per-1-token USD**.
  * Those USD mid-prices do **not** trigger new quotes unless the user requests one.

# UX & safety details

* **Wrap/Unwrap**: ETH↔WETH handled as special cases.
* **Approvals**: Auto-approve ERC-20 (max) when needed.
* **Slippage**: User-set BPS → `amountOutMin`.
* **Decimals**: Always parse/format with the correct token decimals (works for 8/9/18, etc.).
* **Impact**: UI compares the live quote vs. mid USD prices to show price impact.
* **No token-list lock-in**: You can fetch decimals from chain if a token isn’t on the list.

# What you get

* **Any-to-any swap** so long as a path exists via at least one mid (WETH/USDC/etc.).
* **Best execution** across routers and protocols, without forcing everything onto one exchange.
* **Accurate USD display** that doesn’t spam the network (mid prices fetched just on token selection).
* A clean separation between:

  * **execution quotes** (precise, per-amount, best-route search), and
  * **display prices** (per-token mid, lightweight, cached).
