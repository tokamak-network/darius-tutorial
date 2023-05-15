# Bridging ETH with the Optimism SDK

This tutorial teaches you how to use the SDK to transfer ETH between Layer 1 (Goerli) and Layer 2 (Darius).


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

1. Go to [Alchemy](https://www.alchemy.com/) and create two applications:

   - An application on Goerli

   Keep a copy of the two keys.

1. Copy `.env.example` to `.env` and edit it:

   1. Set `MNEMONIC` to point to an account that has ETH on the Goerli test network and the Darius test network.
   1. Set `ALCHEMY_API_KEY` to the key for the Goerli app.

   [This faucet gives ETH on the Goerli network](https://faucet.paradigm.xyz/).

## Run the sample code

The sample code is in `ETHexample.js`, execute it.
After you execute it, wait. It is not unusual for each operation to take minutes on Goerli.

### Expected output

When running on Goerli, the output from the script should be similar to:

```
Deposit ETH
On L1:151154093 Gwei    On L2:139999999 Gwei
Transaction hash (on L1): 0x70d64968fa9e4a58d19c6bdc091ab3e793f1150426168dccf111dbf5b6bee1c4
Waiting for status to change to RELAYED
Time so far 29.92 seconds
On L1:150942902 Gwei    On L2:140000000 Gwei
depositETH took 225.198 seconds


Withdraw ETH
On L1:150942902 Gwei    On L2:140000000 Gwei
Transaction hash (on L2): 0xaddc7562ab0d7125debb9238aeb7f777c2232e724fe30c434a4524a522d4917b

On L1:160107872 Gwei    On L2:130000000 Gwei
withdrawETH took 997.834 seconds
```

As you can see, the total running time is about twenty minutes.


## How does it work?


```js
const ethers = require("ethers")
const optimismSDK = require("@eth-optimism/sdk")
require('dotenv').config()
```

The libraries we need: [`ethers`](https://docs.ethers.io/v5/), [`dotenv`](https://www.npmjs.com/package/dotenv).

```js
const mnemonic = process.env.MNEMONIC
const l1Url = `https://eth-goerli.g.alchemy.com/v2/${process.env.ALCHEMY_API_KEY}`
const l2Url = `https://goerli.optimism.tokamak.network`
```

Configuration, read from `.env`.


```js
// Global variable because we need them almost everywhere
let crossChainMessenger
let addr    // Our address
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



### `setup`

This function sets up the parameters we need for transfers.

```js
const setup = async() => {
  const [l1Signer, l2Signer] = await getSigners()
  addr = l1Signer.address
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

### Variables that make it easier to convert between WEI and ETH

Both ETH and DAI are denominated in units that are 10^18 of their basic unit.
These variables simplify the conversion.

```js
const gwei = 1000000000n
const eth = gwei * gwei   // 10^18
const centieth = eth/100n
```

### `reportBalances`

This function reports the ETH balances of the address on both layers.

```js
const reportBalances = async () => {
  const l1Balance = (await crossChainMessenger.l1Signer.getBalance()).toString().slice(0,-9)
  const l2Balance = (await crossChainMessenger.l2Signer.getBalance()).toString().slice(0,-9)

  console.log(`On L1:${l1Balance} Gwei    On L2:${l2Balance} Gwei`)
}    // reportBalances
```

### `depositETH`

This function shows how to deposit ETH from Goerli to Darius.

```js
const depositETH = async () => {

  console.log("Deposit ETH")
  await reportBalances()
```

To show that the deposit actually happened we show before and after balances.

```js  
  const start = new Date()

  const response = await crossChainMessenger.depositETH(gwei)  
```

`crossChainMessenger.depositETH()` creates and sends the deposit trasaction on L1.

```js
  console.log(`Transaction hash (on L1): ${response.hash}`)
  await response.wait()
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
The `waitForMessageStatus` function does this for us.

The third parameter (which is optional) is a hashed array of options:
- `pollIntervalMs`: The poll interval
- `timeoutMs`: Maximum time to wait

```js
  await reportBalances()    
  console.log(`depositETH took ${(new Date()-start)/1000} seconds\n\n`)
}     // depositETH()
```

Once the message is relayed the balance change on Darius is practically instantaneous.
We can just report the balances and see that the L2 balance rose by 1 gwei.

### `withdrawETH`

This function shows how to withdraw ETH from Darius to Ethereum.

```js
const withdrawETH = async () => { 
  
  console.log("Withdraw ETH")
  const start = new Date()  
  await reportBalances()

  const response = await crossChainMessenger.withdrawETH(centieth)
```

For deposits it was enough to transfer 1 gwei to show that the L2 balance increases.
However, in the case of withdrawals the withdrawing account needs to be pay for finalizing the message, which costs more than that.

By sending 0.01 ETH it is guaranteed that the withdrawal will actually increase the L1 ETH balance instead of decreasing it.

```js
  console.log(`Transaction hash (on L2): ${response.hash}`)
  await response.wait()
  await reportBalances()   
  console.log(`withdrawETH took ${(new Date()-start)/1000} seconds\n\n\n`)  
}  // withdrawETH()
```

### `main`

A `main` to run the setup followed by both operations.

```js
const main = async () => {    
    await setup()
    await depositETH()
    await withdrawETH() 
}  // main



main().then(() => process.exit(0))
  .catch((error) => {
    console.error(error)
    process.exit(1)
  })
```

## Conclusion

You should now be able to write applications that use our SDK and bridge to transfer ETH between layer 1 and layer 2. 

