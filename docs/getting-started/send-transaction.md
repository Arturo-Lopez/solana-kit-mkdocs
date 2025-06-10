---
title: Send a transaction
descrition: Send a transaction to the blockchain and wait for its confirmation
---

In the previous article, we constructed a signed transaction that creates a Solana token. Now, let's see how we can send this transaction and ensure it has been confirmed by the network.

## Send transactions without confirming

The most common way to send transactions is via the `sendTransaction` RPC method. This method accepts an encoded signed transaction and forwards it to the configured RPC node so it can handle this for us.

One way to tackle this would be to encode the transaction ourselves using `getBase64EncodedWireTransaction` and pass it to the `sendTransaction` RPC method like so.

```ts
import { getBase64EncodedWireTransaction } from "@solana/kit";

const encodedTransaction = getBase64EncodedWireTransaction(signedTransaction);
await rpc
  .sendTransaction(encodedTransaction, {
    preflightCommitment: "confirmed",
    encoding: "base64",
  })
  .send();
```

However, Kit offers a helper function that does that for us whilst providing some sensible default values. This function is called `sendTransactionWithoutConfirmingFactory` and, given an RPC object, it returns a function that sends transactions without waiting for confirmation.

```ts
import { sendTransactionWithoutConfirmingFactory } from "@solana/kit";

const sendTransaction = sendTransactionWithoutConfirmingFactory({ rpc });
await sendTransaction(signedTransaction, { commitment: "confirmed" });
```

## Confirmation strategies

Sending transactions is one thing but, more often than not, we'll want to make sure they have been processed by the network before we continue with our application logic.

There are several ways to ensure a transaction was confirmed by the network. For instance, we could poll the network at regular intervals to check the status of the transaction until it has the required commitment level. Another way would be to listen for network events and use them to determine the status of the sent transaction.

Sometimes it may be useful to dig deeper into confirmation strategies and create custom ones that make the most sense for our applications. But for all other cases, Kit provides helpers that abstract this complexity away from us. Namely, it provides two helper functions that both send and confirm transactions. Which function to use depends on the lifetime set on the transaction.

- `sendAndConfirmTransactionFactory` is used for transactions using a blockhash strategy for its lifetime.
- `sendAndConfirmDurableNonceTransactionFactory` is used for transactions using a durable nonce strategy for its lifetime.

Both accept RPC and RPC Subscriptions objects and return a function that sends and confirms transactions for us. Since we are using the blockhash strategy in this tutorial, we'll use the `sendAndConfirmTransactionFactory` function.

```ts
import { sendAndConfirmTransactionFactory } from "@solana/kit";

const sendAndConfirmTransaction = sendAndConfirmTransactionFactory({
  rpc,
  rpcSubscriptions,
});
await sendAndConfirmTransaction(signedTransaction, { commitment: "confirmed" });
```

## Transaction signatures

If you hover on top of the `sendTransaction` or `sendAndConfirmTransaction` functions above, you'll notice that their return type is simply `Promise<void>`. That is, they do not return the transaction signature that uniquely identifies that transaction on the network.

This is because there is a common misconception that one must wait for the transaction to be sent to the network before obtaining its signature. However, that's not the case. The transaction signature is accessible as soon as the transaction is signed by its fee payer. This is because the signature that uniquely identifies a transaction is none other than the fee payer's signature for that transaction.

As such, Kit decouples these two distinct concepts by offering a `getSignatureFromTransaction` function that returns the signature of a transaction that can be used even before it is sent to the network.

```ts
import {
  getSignatureFromTransaction,
  sendAndConfirmTransactionFactory,
} from "@solana/kit";

// Access the transaction signature.
const signature = getSignatureFromTransaction(signedTransaction);

// Send and confirm the transaction.
const sendAndConfirmTransaction = sendAndConfirmTransactionFactory({
  rpc,
  rpcSubscriptions,
});
await sendAndConfirmTransaction(signedTransaction, { commitment: "confirmed" });
```

## Update `client.ts`

With all that new knowledge in hand, let's update our `Client` object to include the new `sendAndConfirmTransaction` function created from the `sendAndConfirmTransactionFactory` helper.

```diff title="src/clien.ts"
  import {
+   sendAndConfirmTransactionFactory,
    // ...
  } from "@solana/kit";

  export type Client = {
    estimateAndSetComputeUnitLimit: <T extends CompilableTransactionMessage>(transactionMessage: T) => Promise<T>;
    rpc: Rpc<SolanaRpcApi>;
    rpcSubscriptions: RpcSubscriptions<SolanaRpcSubscriptionsApi>;
+   sendAndConfirmTransaction: ReturnType<typeof sendAndConfirmTransactionFactory>;
    wallet: TransactionSigner & MessageSigner;
  };

  let client: Client | undefined;
  export async function createClient(): Promise<Client> {
    if (!client) {
      // ...

+     // Create a function to send and confirm transactions.
+     const sendAndConfirmTransaction = sendAndConfirmTransactionFactory({ rpc, rpcSubscriptions });

      // Store the client.
      client = {
        estimateAndSetComputeUnitLimit,
        rpc,
        rpcSubscriptions,
+       sendAndConfirmTransaction,
        wallet,
      };
    }
    return client;
  }
```

## Update `create-mint.ts`

Now, let's make use of our new `client.sendAndConfirmTransaction` helper in the `createMint` function.

All that's left to do is return the address of the `Mint` account we created so the caller of this function can use it for further operations.

And with that, our `createMint` function is complete!

```diff title="src/create-mint.ts"
  import {
    appendTransactionMessageInstructions,
    createTransactionMessage,
    generateKeyPairSigner,
    pipe,
    setTransactionMessageFeePayerSigner,
    setTransactionMessageLifetimeUsingBlockhash,
    signTransactionMessageWithSigners,
  } from "@solana/kit";
  import { getCreateAccountInstruction } from "@solana-program/system";
  import { getInitializeMintInstruction, getMintSize, TOKEN_PROGRAM_ADDRESS } from "@solana-program/token";

  import type { Client } from "./client";

  export async function createMint(client: Client, options: { decimals?: number } = {}) {
    // Prepare inputs.
    const mintSize = getMintSize();
    const [mint, mintRent, { value: latestBlockhash }] = await Promise.all([
      generateKeyPairSigner(),
      client.rpc.getMinimumBalanceForRentExemption(BigInt(mintSize)).send(),
      client.rpc.getLatestBlockhash().send(),
    ]);

    // Build instructions.
    const createAccountIx = getCreateAccountInstruction({
      payer: client.wallet,
      newAccount: mint,
      space: mintSize,
      lamports: mintRent,
      programAddress: TOKEN_PROGRAM_ADDRESS,
    });
    const initializeMintIx = getInitializeMintInstruction({
      mint: mint.address,
      decimals: options.decimals ?? 0,
      mintAuthority: client.wallet.address,
      freezeAuthority: client.wallet.address,
    });

    // Build the transaction message.
    const transactionMessage = await pipe(
      createTransactionMessage({ version: 0 }),
      (tx) => setTransactionMessageFeePayerSigner(client.wallet, tx),
      (tx) => setTransactionMessageLifetimeUsingBlockhash(latestBlockhash, tx),
      (tx) => appendTransactionMessageInstructions([createAccountIx, initializeMintIx], tx),
      (tx) => client.estimateAndSetComputeUnitLimit(tx),
    );

    // Compile the transaction message and sign it.
    const signedTransaction = await signTransactionMessageWithSigners(transactionMessage);

    // Send the transaction and wait for confirmation.
+   await client.sendAndConfirmTransaction(signedTransaction, { commitment: "confirmed" });

+   // Return the address of the created mint account.
+   return mint.address;
  }
```

## Update `index.ts`

Finally, let's update our main `tutorial` function to execute the `createMint` function and log the address of the newly created mint account.

```ts title="src/index.ts"
import { createClient } from "./client";
import { createMint } from "./create-mint";

async function tutorial() {
  const client = await createClient();
  const mintAddress = await createMint(client, { decimals: 2 });
  console.log(`Mint created: ${mintAddress}.`);
}
```

On the next article, we'll use that address to fetch and decode the mint account we just created.
