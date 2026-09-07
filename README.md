# AntiSnipeRamp

**Opens a pool at a punitive fee that decays to normal over a fixed window, and pays every cent of the difference to liquidity providers rather than to the deployer.**

A production Uniswap v4 hook. It prices every swap by overriding the pool's LP fee, so the value it captures is paid to in-range liquidity and never to the hook. No owner, no pause switch, no upgrade path.

- **Site:** https://anti-snipe-ramp.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/AntiSnipeRampHook.sol`](src/hooks/AntiSnipeRampHook.sol)
- **Licence:** Apache-2.0

## How it works

The first block of a new pool is the most valuable block it will ever have. A bot that buys in it and sells an hour later takes the entire launch premium, and everyone who arrives through the front door pays for it. The usual answers are a whitelist, which is a promise rather than a mechanism, or a bonding curve that hands the premium to the deployer, which moves the extraction rather than removing it.

This hook makes the first block expensive to trade in and lets that expense decay: fee(t) = startFee - (startFee - endFee) * min(t, rampSeconds) / rampSeconds A sniper in the first second pays `startFee`, which can be set high enough that the trade is not worth making. A buyer twenty minutes later pays close to `endFee`. Because the fee is an LP fee, the premium the early trader surrenders is paid to the people who put the liquidity up, not to whoever deployed the token.

There is no address in this contract that can receive anything. A second lever handles the case where the fee alone is not enough. While the ramp is running, a single swap may not exceed `maxSwapDuringRamp` units of the specified currency.

This is a size cap, not an identity check, and it is deliberately not per-address: a hook sees the router that called the `PoolManager`, not the person behind it, so any per-address limit is a limit on routers and is defeated by a fresh key. Capping size is enforceable against everyone equally, including the deployer. Set it to zero to disable it.

Prior art: liquidity bootstrapping pools ramp the *price* down and were built for price discovery; several launchpad hooks charge a launch fee and route it to a creator or a protocol treasury. Ramping the *fee* down while directing the proceeds to liquidity is a different mechanism with a different beneficiary, and it composes with any curve rather than replacing it.

## Prior art

Liquidity bootstrapping pools ramp the price down and were built for price discovery; several launchpad hooks charge a launch fee and route it to a creator or a treasury. Ramping the fee down while directing the proceeds to liquidity is a different mechanism with a different beneficiary, and it composes with any curve rather than replacing it.

## Where it does not help

The size cap is per swap, not per address: a hook sees the router that called the PoolManager, not the person behind it, so a determined buyer can split across transactions. The cap raises the cost of sniping rather than preventing it, and the fee ramp is what does the real work.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    AntiSnipeRampHook.Config({
        startFee: /* uint24 */ 0,
        endFee: /* uint24 */ 0,
        rampSeconds: /* uint32 */ 0,
        maxSwapDuringRamp: /* uint128 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```

The pool's `fee` field must be `LPFeeLibrary.DYNAMIC_FEE_FLAG`. The hook rejects a pool initialized without it.

### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `startFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `endFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `rampSeconds` | `uint32` | seconds |
| `maxSwapDuringRamp` | `uint128` | seconds |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `FeeTooLarge(uint24)` | A fee was configured above the protocol maximum of 100%. |
| `InvalidConfig()` | `rampSeconds` was zero, or `endFee` was above `startFee`, which would ramp the fee upward. |
| `NotDynamicFee()` | The hook was attempted to be initialized with a non-dynamic fee. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `SwapTooLargeDuringRamp(uint256,uint128)` | The swap is larger than the pool allows while its launch ramp is running. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `beforeSwap`

Mask: `0x1080`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # AntiSnipeRamp
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # launch, anti-snipe, dynamic-fee, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/anti-snipe-ramp
cd anti-snipe-ramp
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
