# Getting started developing for Darius L2 Testnet

This tutorial teaches you the basics of Darius L2 Testnet development.
Darius L2 Testnet is EVM equivalent, meaning we run a slightly modified version of the same `geth` you run on mainnet.
Therefore, we the differences between Darius L2 Testnet development and Ethereum development are minor.
But a few differences do exist

## Darius L2 Testnet endpoint URL

To access any Ethereum type network you need an endpoint. 
Darius L2 Testnet endpoint is https://goerli.optimism.tokamak.network. Goerli Testnet is used as L1 for Darius L2 Testnet.

### Network choice

For development purposes we recommend you use either a local development node or [Darius L2 Testnet](https://goerli.explorer.tokamak.network/).
The tests examples below use Darius L2 Testnet.


## Interacting with Darius contracts

We have [Hardhat's Greeter contract](https://github.com/tokamak-network/darius-test/blob/main/contracts/Greeter.sol) on Darius L2 Testnet, at address [0x106941459A8768f5A92b770e280555FAF817576f](https://goerli.explorer.tokamak.network/address/0xDe6b80f4700C2148Ba2aF81640a23E153C007C7F/contracts#address-tabs). 
You can verify your development stack configuration by interacting with it. 

As you can see in the different development stacks below, the way you deploy contracts and interact with them on Darius L2 Testnet is almost identical to the way you do it with L1 Ethereum.
The most visible difference is that you have to specify a different endpoint (of course). 

## Hardhat

In [Hardhat](https://hardhat.org/) you use a configuration similar to [this one](https://github.com/tokamak-network/darius-test).

### Connecting to Darius

Follow these steps to add Darius L2 Testnet support to an existing Hardhat project. 


1. Define your network configuration in `.env`:

   ```sh
   # Put the mnemonic for an account on Optimism here
   MNEMONIC=test test test test test test test test test test test junk

   # API KEY for Alchemy
   ALCHEMY_API_KEY=

   # this account is going to be used to call (if not use mnemonic)
   PRIVATE_KEY=
   ```

1. Add `dotenv` to your project:

   ```sh
   yarn add dotenv
   ```

1. Edit `hardhat.config.js`:

   1. Use `.env` for your blockchain configuration:

      ```js
      require('dotenv').config()
      ```

   1. Get the correct URL from the configuration:

      ```js
      const optimismGoerliUrl =
         `https://goerli.optimism.tokamak.network`
      ```


   1. Add a network definition in `module.exports.networks`:

   ```js
   "tokamak-goerli": {
      url: optimismGoerliUrl,
      accounts: { mnemonic: process.env.MNEMONIC }
   }   
   ```

   If you want to test with your personal account, set it as follows in `module.exports.networks`: 

   ```js
   "tokamak-darius-goerli" : {
      url: optimismGoerliUrl,
      accounts: [`${process.env.PRIVATE_KEY}`]
   }
   ```    


### Greeter interaction

1. Run the console:
   ```sh
   cd hardhat
   yarn
   yarn hardhat console --network tokamak-goerli
   ```

   **personal account:**

   Replace the final command with

   ```sh
   yarn hardhat console --network tokamak-darius-goerli
   ```

1. Connect to the Greeter contract:   

   ```js
   Greeter = await ethers.getContractFactory("Greeter")
   greeter = await Greeter.attach("0xDe6b80f4700C2148Ba2aF81640a23E153C007C7F")
   ```   

1. Read information from the contract:

   ```js
   await greeter.greet()
   ```

1. Submit a transaction, wait for it to be processed, and see that it affected the state.

   ```js
   tx = await greeter.setGreeting(`Hardhat: Hello ${new Date()}`)
   rcpt = await tx.wait()  
   await greeter.greet()
   ```

### Deploying a contract

To deploy a contract from the Hardhat console:

```
Greeter = await ethers.getContractFactory("Greeter")
greeter = await Greeter.deploy("Greeter from hardhat")
console.log(`Contract address: ${greeter.address}`)
await greeter.greet()
```

## Best practices

It is best to start development with the EVM provided by the development stack. 
Not only is it faster, but such EVMs often have extra features, such as the [ability to log messages from Solidity](https://hardhat.org/tutorial/debugging-with-hardhat-network.html) or a [graphical user interface](https://trufflesuite.com/ganache/).

After you are done with that development, debug your decentralized application using either a development node or the Goerli test network. 
This lets you debug parts that that are Darius specific such as calls to bridges to transfer assets between layers.

Only when you have a version that works well on a test network should you deploy to the production network, where every transaction has a cost.
