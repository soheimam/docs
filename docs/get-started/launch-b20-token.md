---
title: "Launch a B20 Token"
description: "Spin up a fully configured ERC-20 superset token on Base through the singleton B20 Factory precompile. Set the admin, minter, and supply cap, mint your initial supply, and confirm it on-chain, all without writing or deploying any Solidity."
---

B20 is an ERC-20 superset that runs as a native precompile on Base. You don't write or deploy a token contract. Instead, you ask the singleton **B20 Factory** to create one, fully configured, in a single transaction.

By the end you have a live B20 Asset token (with an admin, a minter, and a supply cap) holding minted supply you can view in the explorer. To accept it as payment in an app, continue with [Accept B20 payments](/apps/guides/accept-b20-payments).

## Before you begin

You need:

* [Foundry](https://getfoundry.sh) installed. You'll use `cast` from it.
* **Base's `forge` build** for deploying. A B20 token is a native precompile, so the factory address holds no contract bytecode. Stock `forge` can't simulate a call to it and aborts a deploy script with `call to non-contract address`. Base's [`base-anvil`](https://github.com/base/base-anvil) fork of `forge` registers the precompiles into its EVM. Build it once and put it ahead of stock `forge` on your `PATH`:

  ```bash theme={null}
  git clone https://github.com/base/base-anvil && cd base-anvil
  cargo build --release -p forge   # produces target/release/forge
  ```
* The network details and a funded account. This tutorial targets **Base Vibenet**, the testnet that hosts the B20 precompiles:

  | Setting | Value |
  |---|---|
  | Network | Base Vibenet |
  | RPC URL | `https://rpc.vibes.base.org/` |
  | Chain ID | `84538453` |
  | Faucet | `https://faucet.vibes.base.org/` |
  | Explorer | _(view the faucet site → Explorer)_ |

Create a `.env` in your project (and add `.env` to `.gitignore`):

```bash .env theme={null}
RPC_URL="https://rpc.vibes.base.org/"
# One account plays deployer + admin + minter for this quickstart.
PRIVATE_KEY="0x..."
ACCOUNT_ADDRESS="0x..."
```

<Note>
If you don't have an account, `cast wallet new` prints a fresh address and key. Put the key in `PRIVATE_KEY` and the address in `ACCOUNT_ADDRESS`.
</Note>

Request testnet ETH for `ACCOUNT_ADDRESS` from the faucet above, then confirm it arrived on Vibenet (not another chain: the address is the same everywhere, but balances are per chain):

```bash theme={null}
source .env
cast balance $ACCOUNT_ADDRESS --rpc-url $RPC_URL
```

<Check>
The command prints a non-zero balance. You're funding one account here. It signs the deploy and the mint, and receives the minted supply. Production setups can separate the admin, minter, and recipient, but this guide doesn't require that split.
</Check>

## Set up your project

```bash theme={null}
forge init b20-quickstart && cd b20-quickstart
forge install base/base-std
```

Add the remappings and the `base = true` flag to `foundry.toml` (under `[profile.default]`). `base = true` tells Base's `forge` build to run the B20 precompiles inside its EVM, so the deploy script's local simulation can call the factory:

```toml foundry.toml theme={null}
base = true
remappings = [
    "base-std/=lib/base-std/src/",
    "base-std-test/=lib/base-std/test/",
]
```

<Note>
The interfaces compile with any Solidity `>=0.8.20 <0.9.0`.
</Note>

## Create your token

The factory's single entry point is `createB20(variant, salt, params, initCalls)`:

* `variant`: `ASSET` or `STABLECOIN`. This guide uses `ASSET`.
* `salt`: caller-chosen entropy that fixes the deterministic token address.
* `params`: ABI-encoded name, symbol, initial admin, and decimals.
* `initCalls`: optional batch of config calls applied at creation.

<Steps>
<Step title="Write the create script">
  Use `B20FactoryLib` to encode `params` and `initCalls`. Create `script/CreateToken.s.sol`:

  ```solidity script/CreateToken.s.sol theme={null}
  // SPDX-License-Identifier: MIT
  pragma solidity ^0.8.20;

  import {Script, console} from "forge-std/Script.sol";

  import {B20Constants} from "base-std/lib/B20Constants.sol";
  import {B20FactoryLib} from "base-std/lib/B20FactoryLib.sol";
  import {IB20Factory} from "base-std/interfaces/IB20Factory.sol";
  import {StdPrecompiles} from "base-std/StdPrecompiles.sol";

  contract CreateToken is Script {
      function run() external returns (address token) {
          // For the quickstart, one account is admin + minter.
          address account = vm.envAddress("ACCOUNT_ADDRESS");
          bytes32 salt = keccak256("my-first-b20");

          // Name, symbol, initial DEFAULT_ADMIN_ROLE holder, decimals (6-18).
          bytes memory params = B20FactoryLib.encodeAssetCreateParams("My Token", "MYT", account, 18);

          // Configuration applied atomically at creation.
          bytes[] memory initCalls = new bytes[](2);
          initCalls[0] = B20FactoryLib.encodeGrantRole(B20Constants.MINT_ROLE, account);
          initCalls[1] = B20FactoryLib.encodeUpdateSupplyCap(1_000_000e18);

          vm.startBroadcast();
          token = StdPrecompiles.B20_FACTORY.createB20(IB20Factory.B20Variant.ASSET, salt, params, initCalls);
          vm.stopBroadcast();

          console.log("B20 token created at:", token);
      }
  }
  ```

  <Warning>
  **Encode with `B20FactoryLib`.** The native implementation rejects non-canonical calldata with `AbiDecodeFailed`; the helpers produce canonical encoding.
  </Warning>

  <Info>
  Asset decimals are fixed at creation and must be in `[6, 18]`. The supply cap is optional; the no-cap sentinel is `type(uint128).max` (the cap can never exceed `uint128.max`).
  </Info>
</Step>

<Step title="Deploy the factory call">
  ```bash theme={null}
  source .env
  forge script script/CreateToken.s.sol --rpc-url $RPC_URL --private-key $PRIVATE_KEY --broadcast
  ```

  On success the script logs the new token's address. Note the `0xB200...` prefix: the factory itself is `0xB20f...`, and the tokens it creates start `0xB200...`:

  ```text theme={null}
  == Logs ==
    B20 token created at: 0xB200...
  ```
</Step>

<Step title="Capture the token address">
  Save it into your environment so the next step needs no copy-paste. The broadcast artifact holds the return value:

  ```bash theme={null}
  export TOKEN_ADDRESS=$(jq -r '.returns.token.value' \
    broadcast/CreateToken.s.sol/84538453/run-latest.json)
  echo "TOKEN_ADDRESS=$TOKEN_ADDRESS"
  ```
</Step>
</Steps>

## Mint and verify

Minting requires `MINT_ROLE`, which `initCalls` granted to your account.

<Steps>
<Step title="Mint supply">
  ```bash theme={null}
  cast send $TOKEN_ADDRESS "mint(address,uint256)" $ACCOUNT_ADDRESS 1000e18 \
    --rpc-url $RPC_URL --private-key $PRIVATE_KEY
  ```

  `cast send` prints a receipt with `status 1 (success)`.
</Step>

<Step title="Confirm the balance">
  ```bash theme={null}
  cast call $TOKEN_ADDRESS "balanceOf(address)(uint256)" $ACCOUNT_ADDRESS --rpc-url $RPC_URL
  # 1000000000000000000000 [1e21]
  ```

  <Check>
  The token now holds minted supply on-chain. Open the explorer (from the faucet site) and search `$TOKEN_ADDRESS` to view it.
  </Check>
</Step>
</Steps>

## What you built

In this guide you:

* created a B20 Asset token with one `createB20` call,
* configured its admin, minter, and supply cap atomically via `initCalls`,
* minted supply, and
* verified the balance on-chain.

You did all of this without writing, deploying, or auditing a token contract.

## Next steps

<Card title="Accept B20 payments in an app" icon="credit-card" href="/apps/guides/accept-b20-payments">
Wire this token into a checkout flow that tags each payment with an order ID and reconciles it from on-chain events.
</Card>

* Gate transfers or mints with PolicyRegistry policies, add granular pause, or manage roles. See the B20 overview.
* Issue a stablecoin variant (fixed 6 decimals, immutable currency code).

<AccordionGroup>
<Accordion title="Why B20 instead of a standard ERC-20?">
With a standard ERC-20 you write (or fork) a token contract, audit it, deploy it, and add access control, pausing, and compliance hooks yourself. With B20, that logic is part of the chain. Every token the factory creates runs the same native implementation, with roles, supply cap, pause, policy gating, memos, and `permit` built in at full ERC-20 selector parity. Tooling that targets the ERC-20 interface works with it unchanged.
</Accordion>
</AccordionGroup>
