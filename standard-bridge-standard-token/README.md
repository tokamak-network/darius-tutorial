# Bridging your Standard ERC20 token to Darius using the Standard Bridge

We will show you an example of how to distribute and use L1 standard token and L2 standard tokens. The current github explains, and the github with actual contracts is [`contract-tutorial`](https://github.com/tokamak-network/tokamak-optimism-test) here(You can run the actual test code here on Github).

## Deploying a standard token on Goerli

If you have a token in Layer 1, you can proceed from the next step. If not, you need to distribute the tokens on Layer 1 first.

Execute `scripts/l1Token-deploy.js` to deploy the token to Layer 1.

If the environment is goerli, you can deploy to goerli with the following command.

```
npx hardhat run scripts/l1Token-deploy.js --network goerli
```

When distribution is complete, the address of the distributed token is known and entered as `L1_TOKEN_ADDRESS` in the .env environment.


## Deploying a standard token on Darius

Let's proceed with distributing standard Token to L2 by opening a terminal.

```
yarn hardhat console --network tokamak-optimism-goerli
```

Entering the above command will open a terminal for that network.
Tokens can be distributed to L2 with the following command.

```
l2StandardERC20Factory = await ethers.getContractFactory("L2StandardERC20")
l2StandardERC20 = await l2StandardERC20Factory.deploy(
   "0x4200000000000000000000000000000000000010",
   process.env.L1_CUSTOM_ADDRESS,
   process.env.L2_TOKEN_NAME,
   process.env.L2_TOKEN_SYMBOL)
```

Distribution to L2 has been completed with the above command, and we will check the address of the L2 token.

```
l2StandardERC20.address
```

you can use to instantiate `L2StandardERC20` either on a local dev node or on `tokamak-optimism-goerli`.

### Configuration

1. Install the necessary packages.

   ```sh
   yarn
   ```

1. Copy the example configuration to the production one:

   ```sh
   cp .env.example .env
   ```

1. Get an [Alchemy](https://dashboard.alchemyapi.io/) application for Optimism, either Optimism Goerli for testing or Optimism Mainnet for deployment.

1. Edit `.env` to set the deployment parameters:

   - `PRIVATE_KEY`, this account is going to be used to call the factory and create your L2 ERC20. Remember to fund your account for deployment.
   - `ALCHEMY_API_KEY`, the key for the alchemy application for the endpoint.
   - `L1_TOKEN_ADDRESS`, the address of the L1 ERC20 which you want to bridge.
   - `L2_TOKEN_NAME` and `L2_TOKEN_SYMBOL` are the parameters for the L2 token contract. 
     In almost all cases, these would be the same as the L1 token name and symbol.

### Running the deploy script

1. Run the script:

   ```sh
   yarn hardhat run scripts/deploy-standard-token.js --network tokamak-optimism-goerli
   ```

The script uses our token factory contract `OVM_L2StandardTokenFactory` available as a predeploy at `0x4200000000000000000000000000000000000012` to deploy a standard token on L2. 
At the end you should get a successful output with the text for a `data.json` file you'll be able to use to [add the token to the bridge](https://github.com/ethereum-optimism/ethereum-optimism.github.io).
Note that if you have the token both on the test network and the production network you should not use that `data.json` by itself, but combine it the information in the two files. 

## Deploying a Custom Token

When the `L2StandardERC20` implementation does not satisfy your requirements, we can consider allowing a custom implemetation. 
See this [tutorial on getting a custom token implemented and deployed](../standard-bridge-custom-token/README.md) to Optimistic Ethereum.

## Testing 

For testing your token, see [tutorial on depositing and withdrawing between L1 and L2](../cross-dom-bridge-erc20).

