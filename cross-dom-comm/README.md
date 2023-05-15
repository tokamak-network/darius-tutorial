# Communication between contracts on Darius and Goerli

This tutorial teaches you how to do interlayer communication.
You will learn how run a contract on Ethereum that runs another contract on Optimism, and also how to run a contract on Darius that calls a contract on Goerli.

## Test Code git

[Github](https://github.com/tokamak-network/darius-test/tree/main/contracts)

## Seeing it in action

To show how this works we installed [a slightly modified version of HardHat's `Greeter.sol`](https://github.com/tokamak-network/darius-test/blob/main/contracts/Greeter.sol) on both L1 Goerli and L2 Darius.


| Network | Greeter address  |
| ------- | ---------------- |
| Goerli (L1) | [0x51aB33d511a74aBeFDce2d4AddB92991B73F8937](https://goerli.etherscan.io/address/0x51aB33d511a74aBeFDce2d4AddB92991B73F8937) |
| Darius (L2) | [0xDe6b80f4700C2148Ba2aF81640a23E153C007C7F](https://goerli.explorer.tokamak.network/address/0xDe6b80f4700C2148Ba2aF81640a23E153C007C7F) |

::: Tip What if somebody else uses the same contracts at the same time?
If somebody else uses these contracts while you are going through the tutorial, they might update the greeting after you.
In that case you'll see the wrong greeting when you call the `Greeter` contract.
However, you can still verify your controller works in one of these ways:

- Find the transaction on either [Goerli Etherscan](https://goerli.etherscan.io/address/0x7fA4D972bB15B71358da2D937E4A830A9084cf2e#internaltx) or [Darius Block Explorer](https://goerli.explorer.tokamak.network/address/0xDe6b80f4700C2148Ba2aF81640a23E153C007C7F/internal-transactions#address-tabs).
  In either case, it will be an internal transaction because the contract called directly is the cross domain messenger.
- Just try again.
:::

### Hardhat

This is how you can see communication between domains work in hardhat.

#### Setup

This setup assumes you already have [Node.js](https://nodejs.org/en/) and [yarn](https://classic.yarnpkg.com/) installed on your system. 

1. Go to [Alchemy](https://www.alchemy.com/) and create one applications:

   - An application on Goerli

   Keep a copy of the one key.

1. Copy `.env.example` to `.env` and edit it:

   1. Set `MNEMONIC` to point to an account that has ETH on the Goerli test network and the Darius test network.
   1. Set `ALCHEMY_API_KEY` to the key for the Goerli app.
   
1. Install the necessary packages.

   ```sh
   yarn
   ```

#### Goerli message to Darius

1. Connect the Hardhat console to Darius(L2):

   ```sh
   yarn hardhat console --network tokamak-darius-goerli
   ```

1. Connect to the greeter on L2:
  
   ```js
   Greeter = await ethers.getContractFactory("Greeter")
   greeter = await Greeter.attach("0xDe6b80f4700C2148Ba2aF81640a23E153C007C7F")
   await greeter.greet()
   ```

1. In a separatate terminal window connect the Hardhat console to Goerli (L1):

   ```sh
   yarn hardhat console --network goerli
   ```

1. Deploy and call the `FromL1_ControlL2Greeter` contract.

   ```js
   Controller = await ethers.getContractFactory("FromL1_ControlL2Greeter")
   controller = await Controller.deploy()
   tx = await controller.setGreeting(`Hello from L1 ${Date()}`)
   rcpt = await tx.wait()
   ```

1. Make a note of the address of the L1 controller.

   ```js
   controller.address
   ```

1. Back in the L2 console, see the new greeting.
   Note that it may take a few minutes to update after the transaction is processed on L1.

   ```sh
   yarn hardhat console --network tokamak-darius-goerli
   ```

   ```js
   Greeter = await ethers.getContractFactory("Greeter")
   greeter = await Greeter.attach("0x51aB33d511a74aBeFDce2d4AddB92991B73F8937")
   await greeter.greet()
   ```

#### Darius message to Goerli

1. Get the current L1 greeting. There are two ways to do that:

   - [Browse to the Greeter contract on Etherscan](https://goerli.etherscan.io/address/0x51aB33d511a74aBeFDce2d4AddB92991B73F8937#readContract) and click **greet** to see the greeting.

   - Run these commands in the Hardhat console connected to L1 Goerli:

     ```js
     Greeter = await ethers.getContractFactory("Greeter")
     greeter = await Greeter.attach("0x51aB33d511a74aBeFDce2d4AddB92991B73F8937")
     await greeter.greet()     
     ```

1. Connect the Hardhat console to Darius (L2):

   ```sh
   yarn hardhat console --network tokamak-darius-goerli
   ```

1. Deploy and call the `FromL2_ControlL1Greeter` contract.

   ```js
   Controller = await ethers.getContractFactory("FromL2_ControlL1Greeter")
   controller = await Controller.deploy()
   tx = await controller.setGreeting(`Hello from L2 ${Date()}`)
   rcpt = await tx.wait()
   ```

1. Make a note of the address of `FromL2_ControlL1Greeter`.

   ```js
   controller.address
   ```

1. Back in the L1 console, see the new greeting.
   Note that it may take a few minutes to update after the transaction is processed on L2.

   ```sh
   yarn hardhat console --network goerli
   ```

   ```js
   Greeter = await ethers.getContractFactory("Greeter")
   greeter = await Greeter.attach("0xDe6b80f4700C2148Ba2aF81640a23E153C007C7F")
   await greeter.greet()
   ```

## How it's done (in Solidity)

We'll go over the L1 contract that controls Greeter on L2, [`FromL1_ControlL2Greeter.sol`](https://github.com/tokamak-network/darius-test/blob/main/contracts/FromL1_ControlL2Greeter.sol).
Except for addresses, the contract going the other direction, [`FromL2_ControlL1Greeter.sol`](https://github.com/tokamak-network/darius-test/blob/main/contracts/FromL2_ControlL1Greeter.sol), is identical.

```solidity
//SPDX-License-Identifier: Unlicense
// This contracts runs on L1, and controls a Greeter on L2.
pragma solidity ^0.8.0;

import { ICrossDomainMessenger } from 
    "@eth-optimism/contracts/libraries/bridge/ICrossDomainMessenger.sol";
```

This line imports the interface to send messages, [`ICrossDomainMessenger.sol`](https://github.com/tokamak-network/darius-test/blob/main/contracts/libraries/bridge/ICrossDomainMessenger.sol).


```solidity
contract FromL1_ControlL2Greeter {
    address crossDomainMessengerAddr = 0x2878373BA3Be0Ef2a93Ba5b3F7210D76cb222e63;
```

To call L2 from L1 on mainnet, you need to `Proxy_OVM_L1CrossDomainMessenger`, 0x2878373BA3Be0Ef2a93Ba5b3F7210D76cb222e63.
To call L1 from L2, on either mainnet or Goerli, use the address of `L2CrossDomainMessenger`, 0x4200000000000000000000000000000000000007.

```solidity
    address greeterL2Addr = 0xDe6b80f4700C2148Ba2aF81640a23E153C007C7F;
```    

This is the address on which `Greeter` is installed on Darius.


```solidity
    function setGreeting(string calldata _greeting) public {
```

This function sets the new greeting. Note that the string is stored in `calldata`. 
This saves us some gas, because when we are called from an externally owned account or a different contract there no need to copy the input string to memory.
The downside is that we cannot call `setGreeting` from within this contract, because contracts cannot modify their own calldata.

```solidity
        bytes memory message;
```

This is where we'll store the message to send to L2.

```solidity 
        message = abi.encodeWithSignature("setGreeting(string)", 
            _greeting);
```

Here we create the message, the calldata to be sent on L2.
The Solidity [`abi.encodeWithSignature`](https://docs.soliditylang.org/en/v0.8.12/units-and-global-variables.html?highlight=abi.encodeWithSignature#abi-encoding-and-decoding-functions) function creates this calldata.
As [specified in the ABI](https://docs.soliditylang.org/en/v0.8.12/abi-spec.html), it is four bytes of signature for the function being called followed by the parameter, in this case a string.

```solidity
        ICrossDomainMessenger(crossDomainMessengerAddr).sendMessage(
            greeterL2Addr,
            message,
            1000000   // within the free gas limit amount
        );
```

This call actually sends the message. It gets three parameters:

1. The address on L2 of the contract being contacted
1. The calldata to send that contract
1. The gas limit.
   As long as the gas limit is below the [`enqueueL2GasPrepaid`](https://etherscan.io/address/0x5E4e65926BA27467555EB562121fac00D24E9dD2#readContract) value, there is no extra cost.
   Note that this parameter is also required on messages from L2 to L1, but there it does not affect anything.

```solidity
    }      // function setGreeting 
}          // contract FromL1_ControlL2Greeter
```


## Getting the source address

If you look at Etherscan, for either the [L1 Greeter](https://goerli.etherscan.io/address/0x51aB33d511a74aBeFDce2d4AddB92991B73F8937) or the [L2 Greeter](https://goerli.explorer.tokamak.network/address/0xDe6b80f4700C2148Ba2aF81640a23E153C007C7F), you will see events with the source address on the other layer.
The way this works is that the cross domain messenger that calls the target contract has a method, `xDomainMessageSender()`, that returns the source address. It is used by the `getXsource` function in `Greeter`.

```solidity
  // Get the cross domain origin, if any
  function getXorig() private view returns (address) {
    address cdmAddr = address(0);    
```

It might look like it would be more efficient to calculate the address of the cross domain messanger just once, but that would involve changing the state, which is an expensive operation. 
Unless we are going to run this code thousands of times, it is more efficient to just have a few `if` statements.

```solidity
    // Goerli
    if (block.chainid == 5)
      cdmAddr = 0x2878373BA3Be0Ef2a93Ba5b3F7210D76cb222e63;

    // L2 Darius
    if (block.chainid == 5050)
      cdmAddr = 0x4200000000000000000000000000000000000007;
      
```

There are three possibilities for the cross domain messenger's address on L1, because the address is not under our control.
On L2 Darius has full control of the genesis block, so we can put all of our contracts on convenient addresses.

```solidity
    // If this isn't a cross domain message
    if (msg.sender != cdmAddr)
      return address(0);
```

If the sender isn't the cross domain messenger, then this isn't a cross domain message.
Just return zero.


```solidity
    // If it is a cross domain message, find out where it is from
    return ICrossDomainMessenger(cdmAddr).xDomainMessageSender();
  }    // getXorig()
```

If it is the cross domain messenger, call `xDomainMessageSender()` to get the original source address.

## Conclusion

You should now be able to control contracts on Darius from Goerli or the other way around.

