---
title: About
descrition: Install and get started with Kit
---

Kit is a JavaScript SDK for building Solana apps across environments like Node, the web, and React Native. It provides a comprehensive set of data types and helper functions, forming the foundation for interacting with Solana in JavaScript.

!!! info "Coming from Web3.js?"

    Check out our [Upgrade guide](http://127.0.0.1:8000/#) to see how Kit compares to Web3.js.

## Quick start

Follow these simple steps to install and get started with Kit:

## Install Kit

Install Kit using the main `@solana/kit library`. It includes many smaller packages — such as `@solana/rpc` or `@solana/transactions` — that can also be used individually for more granular imports.

=== "npm"

    ``` bash
    npm instal --save @solana/kit
    ```

=== "yarn"

    ``` bash
    yarn add @solana/kit
    ```

=== "pnpm"

    ``` bash
    pnpm add @solana/kit
    ```

=== "bun"

    ``` bash
    bun add @solana/kit
    ```

## Install program clients

Install Kit compatible clients for any program you want to interact with. For example, to interact with the System program, install the `@solana-program/system` package. You can find a list of available program clients here.

=== "npm"

    ``` bash
    npm instal --save @solana-program/system \
        @solana-program/memo \
        @solana-program/token \
        @solana-program/compute-budget
    ```

=== "yarn"

    ```sh
    yarn add @solana-program/system \
        @solana-program/memo \
        @solana-program/token \
        @solana-program/compute-budget
    ```

=== "pnpm"

    ```sh
    pnpm add @solana-program/system \
        @solana-program/memo \
        @solana-program/token \
        @solana-program/compute-budget
    ```

=== "bun"

    ```sh
    bun add @solana-program/system \
        @solana-program/memo \
        @solana-program/token \
        @solana-program/compute-budget
    ```

## Set up your project

Use primitives provided by Kit to set up your project and start interacting with Solana!

For example, if you are planning on sending transactions, you will likely need an RPC API, an RPC Subscriptions API and a "send and confirm" strategy.

```ts
import {
  createSolanaRpc,
  createSolanaRpcSubscriptions,
  sendAndConfirmTransactionFactory,
} from "@solana/kit";

const rpc = createSolanaRpc("https://api.devnet.solana.com");
const rpcSubscriptions = createSolanaRpcSubscriptions(
  "wss://api.devnet.solana.com"
);
const sendAndConfirmTransaction = sendAndConfirmTransactionFactory({
  rpc,
  rpcSubscriptions,
});
```
