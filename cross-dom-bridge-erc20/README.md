# Bridging ERC-20 tokens

This tutorial teaches you how to use the SDK to transfer ERC-20 tokens between Layer 1 (Ethereum) and Layer 2 (Tokamak).
While you *could* use the bridge contracts directly, a simple usage error can cause you to lock tokens in the bridge forever and lose their value. 
The SDK provides transparent safety rails to prevent that mistake.


## Setup

1. Ensure your computer has:
   - [`git`](https://git-scm.com/downloads)
   - [`node`](https://nodejs.org/en/)
   - [`yarn`](https://classic.yarnpkg.com/lang/en/docs/install/#mac-stable)

1. Clone this repository and enter it.

   ```sh
   git clone https://github.com/tokamak-network/tokamak-optimism-test.git
   cd tokamak-optimism-test
   ```

1. Install the necessary packages.

   ```sh
   npm install
   ```

1. Go to [Alchemy](https://www.alchemy.com/) and create one application:

   - An application on Goerli

   Keep a copy of the one key.

1. Copy `.env.example` to `.env` and edit it:

   1. Set `MNEMONIC` to point to an account that has ETH on the Goerli test network and the Optimism Goerli test network.
   1. Set `ALCHEMY_API_KEY` to the key for the Goerli app.

   [This faucet gives ETH on the Goerli network](https://faucet.paradigm.xyz/). 

## Run the sample code WithdrawTest

The sample code is in `DepositTest.js` and `WithdrawTest.js`, execute it.
After you execute it, wait. It is not unusual for each operation to take minutes on Goerli.

### Expected output

When running on Goerli, the output from the script should be similar to:

```
Deposit ERC20
Allowance given by tx 0x7c541937bcdb76550aecc4558dd3c53955ea2fa61e38006fa3be246277c5d2c9
	More info: https://goerli.etherscan.io/tx/0x7c541937bcdb76550aecc4558dd3c53955ea2fa61e38006fa3be246277c5d2c9
Time so far 24.749 seconds
before deposit
TON on L1:     TON on L2:401
Deposit transaction hash (on L1): 0xa083b921a583e3eb0a149e79a638cc43aec42af6a80812b5a3883f8ce799a177
	More info: https://goerli.etherscan.io/tx/0xa083b921a583e3eb0a149e79a638cc43aec42af6a80812b5a3883f8ce799a177
Waiting for status to change to RELAYED
Time so far 49.39 seconds
after deposit
TON on L1:999     TON on L2:402
depositERC20 took 230.453 seconds


Withdraw ERC20
before withdraw
TON on L1:999     TON on L2:402
Transaction hash (on L2): 0x1629ab4113b3aa68447a0a08d066c5c24be1214c624b4c622578dd6e20ea05ae
	For more information: https://goerli.explorer.tokamak.network/tx/0x1629ab4113b3aa68447a0a08d066c5c24be1214c624b4c622578dd6e20ea05ae
after withdraw
TON on L1:1000     TON on L2:401
```

As you can see, the total running time is about eight minutes.
It could be longer


## How does it work?


```js
#! /usr/local/bin/node

// Transfers between L1 and L2 using the Optimism SDK

const ethers = require("ethers")
const optimismSDK = require("@eth-optimism/sdk")
require('dotenv').config()

```

The libraries we need: [`ethers`](https://docs.ethers.io/v5/), [`dotenv`](https://www.npmjs.com/package/dotenv) and the Optimism SDK itself.

```js
const mnemonic = process.env.MNEMONIC
const l1Url = `https://eth-goerli.g.alchemy.com/v2/${process.env.ALCHEMY_API_KEY}`
const l2Url = `https://goerli.optimism.tokamak.network`
```

Configuration, read from `.env`.


```js
// Contract addresses for TON tokens, taken
const erc20Addrs = {
  l1Addr: "0x68c1F9620aeC7F2913430aD6daC1bb16D8444F00",
  l2Addr: "0x7c6b91D9Be155A6Db01f749217d76fF02A7227F2"
}    // erc20Addrs
```

The addresses of the ERC-20 token on L1 and L2.

```js
// Global variable because we need them almost everywhere
let crossChainMessenger
let l1ERC20, l2ERC20    // OUTb contracts to show ERC-20 transfers
let ourAddr   // Our address
```

The configuration parameters required for transfers.

### `getSigners`

This function returns the two signers (one for each layer). 

```js
const getSigners = async () => {
    const l1RpcProvider = new ethers.providers.JsonRpcProvider(l1Url)    
    const l2RpcProvider = new ethers.providers.JsonRpcProvider(l2Url)
```

The first step is to create the two providers, each connected to an endpoint in the appropriate layer.

```js
    const hdNode = ethers.utils.HDNode.fromMnemonic(mnemonic)
    const privateKey = hdNode.derivePath(ethers.utils.defaultPath).privateKey
```    

To derive the private key and address from a mnemonic it is not enough to create the `HDNode` ([Hierarchical Deterministic Node](https://en.bitcoin.it/wiki/Deterministic_wallet#Type_2:_Hierarchical_deterministic_wallet)).
The same mnemonic can be used for different blockchains (it's originally a Bitcoin standard), and the node with Ethereum information is under [`ethers.utils.defaultPath`](https://docs.ethers.io/v5/single-page/#/v5/api/utils/hdnode/-%23-hdnodes--defaultpath).

```js    
    const l1Wallet = new ethers.Wallet(privateKey, l1RpcProvider)
    const l2Wallet = new ethers.Wallet(privateKey, l2RpcProvider)

    return [l1Wallet, l2Wallet]
}   // getSigners
```

Finally, create and return the wallets.
We need to use wallets, rather than providers, because we need to sign transactions.

### `erc20ABI`

A fragment of the ABI with the functions we need to call directly.


```js
const erc20ABI = [
  // balanceOf
  {
    constant: true,
    inputs: [{ name: "_owner", type: "address" }],
    name: "balanceOf",
    outputs: [{ name: "balance", type: "uint256" }],
    type: "function",
  },
  // approve
    {
      constant: true,
      inputs: [
        { name: "spender", type: "address" },
        { name: "amount", type: "uint256" }],
      name: "approve",
      outputs: [{ name: "", type: "bool" }],
      type: "function",
    },
```

This is `balanceOf` and `approve` from the ERC-20 standard, used to get the balance of an address and used to approve. 

```js
  // faucet
  {
    inputs: [],
    name: "faucet",
    outputs: [],
    stateMutability: "nonpayable",
    type: "function"
  }  
]    // erc20ABI
```

This is `faucet`, a function supported by the L1 contract, which gives the caller a thousand tokens.
Technically speaking we should have two ABIs, because the L2 contract does not have `faucet`, but that would be a needless complication in this case when we can just avoid trying to call it.


### `setupCrossMessengerAndContract`

This function sets up the parameters we need for transfers.

```js
const setupCrossMessengerAndContract = async() => {
  const [l1Signer, l2Signer] = await getSigners()
  ourAddr= l1Signer.address
```

Get the signers we need, and our address.

```js
  crossChainMessenger = new optimismSDK.CrossChainMessenger({
      l1ChainId: 5,    // Goerli value, 1 for mainnet
      l2ChainId: 420,  // Goerli value, 10 for mainnet
      l1SignerOrProvider: l1Signer,
      l2SignerOrProvider: l2Signer
  })
```

Create the [`CrossChainMessenger`](https://sdk.optimism.io/classes/crosschainmessenger) object that we use to transfer assets.


```js
  l1Contract = new ethers.Contract(erc20Addrs.l1Addr, erc20ABI, l1Signer)
  l2Contract = new ethers.Contract(erc20Addrs.l2Addr, erc20ABI, l2Signer)
}    // setup
```

### `depositERC20`

This function shows how to deposit an ERC-20 token from Ethereum to TokamakLayer2.

```js
const depositAmount = ethers.utils.parseEther("1")
```

`TON` tokens are divided into $1^18$ basic units, same as ETH divided into wei. 

```js
const depositERC20 = async () => {

  console.log("Deposit ERC20")
  tx = (await l1Contract.balanceOf(l1Signer.address)).toString().slice(0,-18)
  tx2 = (await l2Contract.balanceOf(l2Signer.address)).toString().slice(0,-18)
  console.log('before deposit');
  console.log(`TON on L1:${tx}     TON on L2:${tx2}`)
```

To show that the deposit actually happened we show before and after balances.

```js  
  const start = new Date()

  // Need the l2 address to know which bridge is responsible
  depositTx1 = await crossChainMessenger.approveERC20(
    erc20Addrs.l1Addr, erc20Addrs.l2Addr, oneToken)
```

To enable the bridge to transfer ERC-20 tokens, it needs to get an allowance first.
The reason to use the SDK here is that it looks up the bridge address for us.
While most ERC-20 tokens go through the standard bridge, a few require custom business logic that has to be written into the bridge itself.
In those cases there is a custom bridge contract that needs to get the allowance. 

```js
  await depositTx1.wait()
  console.log(`Allowance given by tx ${depositTx1.hash}`)
  console.log(`\tMore info: https://goerli.etherscan.io/tx/${depositTx1.hash}`)
  console.log(`Time so far ${(new Date()-start)/1000} seconds`)
```

Wait until the allowance transaction is processed and then report the time it took and the hash.

```js
  depositTx2 = await crossChainMessenger.depositERC20(
    erc20Addrs.l1Addr, erc20Addrs.l2Addr, oneToken)
```

[`crossChainMessenger.depositERC20()`](https://sdk.optimism.io/classes/crosschainmessenger#depositERC20-2) creates and sends the deposit trasaction on L1.

```js
  console.log(`Deposit transaction hash (on L1): ${depositTx2.hash}`)
  console.log(`\tMore info: https://goerli.etherscan.io/tx/${depositTx2.hash}`)
  await depositTx2.wait()
```

Of course, it takes time for the transaction to actually be processed on L1.

```js
  console.log("Waiting for status to change to RELAYED")
  console.log(`Time so far ${(new Date()-start)/1000} seconds`)
  await crossChainMessenger.waitForMessageStatus(response.hash,
                                                  optimismSDK.MessageStatus.RELAYED)
```

After the transaction is processed on L1 it needs to be picked up by an off-chain service and relayed to L2. 
To show that the deposit actually happened we need to wait until the message is relayed. 
The [`waitForMessageStatus`](https://sdk.optimism.io/classes/crosschainmessenger#waitForMessageStatus) function does this for us.
[Here are the statuses we can specify](https://sdk.optimism.io/enums/messagestatus).

The third parameter (which is optional) is a hashed array of options:
- `pollIntervalMs`: The poll interval
- `timeoutMs`: Maximum time to wait

```js
  console.log('after deposit');
  console.log(`TON on L1:${tx3}     TON on L2:${tx4}`)
  console.log(`depositERC20 took ${(new Date()-start)/1000} seconds\n\n`)
}     // depositERC20()
```

Once the message is relayed the balance change on TokamakLayer2 is practically instantaneous.
We can just report the balances and see that the L2 balance rose by 1 gwei.

### `withdrawERC20`

This function shows how to withdraw ERC-20 from TokamakLayer2 to Ethereum.

```js
const withdrawERC20 = async () => {
  const [l1Signer, l2Signer] = await getSigners()
  console.log("Withdraw ERC20");
  tx = (await l1Contract.balanceOf(l1Signer.address)).toString().slice(0,-18)
  tx2 = (await l2Contract.balanceOf(l2Signer.address)).toString().slice(0,-18)
  console.log('before withdraw');
  console.log(`TON on L1:${tx}     TON on L2:${tx2}`)

  withdrawalTx1 = await crossChainMessenger.withdrawERC20(l1Contract.address, erc20Addrs.l2Addr, withdrawAmount)
  console.log(`\ttransaction hash (on L2): ${withdrawalTx1.hash}`)
  console.log(`\tFor more information: https://goerli.explorer.tokamak.network/tx/${withdrawalTx1.hash}`)
  await withdrawalTx1.wait()

  tx3 = (await l1Contract.balanceOf(l1Signer.address)).toString().slice(0,-18)
  tx4 = (await l2Contract.balanceOf(l2Signer.address)).toString().slice(0,-18)
  console.log('after withdraw');
  console.log(`TON on L1:${tx3}     TON on L2:${tx4}`)
```

To show that the withdraw actually happened we show before and after balances.

### `main`

A `main` to run the setup followed.

`DepositTest`

```js
const main = async () => {
    await setupCrossMessengerAndContract()
    await depositERC20()
}  // main


main().then(() => process.exit(0))
  .catch((error) => {
    console.error(error)
    process.exit(1)
  })
```

`WithdrawTest`

```js
const main = async () => {
    await setupCrossMessengerAndContract()
    await withdrawERC20()
}  // main


main().then(() => process.exit(0))
  .catch((error) => {
    console.error(error)
    process.exit(1)
  })
```

## Conclusion

You should now be able to write applications that use our SDK and bridge to transfer ERC-20 assets between layer 1 and layer 2. 

* [Hop](https://docs.hop.exchange/js-sdk/getting-started)
* [Synapse](https://docs.synapseprotocol.com/bridge-sdk/sdk-reference/bridge-synapsebridge)
* [Across](https://docs.across.to/bridge/developers/across-sdk)
* [Celer Bridge](https://cbridge-docs.celer.network/developer/cbridge-sdk)
