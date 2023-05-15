# Bridging your Custom ERC20 token to Darius using the Standard Bridge

We will show you an example of how to distribute and use L2 custom tokens. The current github explains, and the github with actual contracts is [`contract-tutorial`](https://github.com/tokamak-network/darius-test
) here(You can run the actual test code here on Github).

## Customizing the `L2StandardERC20` implementation

Our example here implements a custom token [`L2CustomERC20`](https://github.com/tokamak-network/darius-test/blob/main/contracts/standards/L2StandardERC20.sol) based on the `L2StandardERC20` but with `8` decimal points, rather than `18`.

The basic role of Token is to import it to do the same as `L2StandardERC20`. This standard token implementation is based on the OpenZeppelin ERC20 contract and implements the required `IL2StandardERC20` interface.

```
import { L2StandardERC20 } from "../standards/L2StandardERC20.sol";
```

Then the only thing we need to do is call the internal `_setupDecimals(8)` method to alter the token `decimals` property from the default `18` to `8`.

## Deploying the Token on L1

If you have a token in Layer 1, you can proceed from the next step. If not, you need to distribute the tokens on Layer 1 first.

Execute `scripts/l1CustomToken-deploy.js` to deploy the token to Layer 1.

If the environment is goerli, you can deploy to goerli with the following command.

```
npx hardhat run scripts/l1CustomToken-deploy.js --network goerli
```

When distribution is complete, the address of the distributed token is known and entered as `L1_CUSTOM_ADDRESS` in the .env environment.

## Deploying the Custom Token on L2

Let's proceed with distributing Custom Token to L2 by opening a terminal.

```
yarn hardhat console --network tokamak-optimism-goerli
```

Entering the above command will open a terminal for that network.
Tokens can be distributed to Darius L2 with the following command.

```
l2CustomERC20Factory = await ethers.getContractFactory("L2CustomERC20")
l2CustomERC20 = await l2CustomERC20Factory.deploy(
   "0x4200000000000000000000000000000000000010",
   process.env.L1_CUSTOM_ADDRESS)
```

Distribution to L2 has been completed with the above command, and we will check the address of the L2 token.

```
l2CustomERC20.address
```

you can use to instantiate `L2CustomERC20` either on a local dev node or on `tokamak-optimism-goerli`.


### Configuration

See an example config at [.env.example](.env.example); copy into a `.env` file before running.

`PRIVATE_KEY` - this account is going to be used to call the factory and create your L2 ERC20. Remember to fund your account for deployment.
`ALCHEMY_API_KEY` - is your ALCHEMY_API for using `goerli` and `mainnet`.
`ETHERSCAN_API_KEY` - is your etherscan API key
`L1_CUSTOM_ADDRESS` - address of the L1 ERC20 which you want to bridge.

### Concluding

If the configuration is properly set and token distribution to Goerli and token distribution to Darius are executed in order, the token distribution is completed successfully.

## Testing 

For testing your token, see [tutorial on depositing and withdrawing between L1 and L2](../cross-dom-bridge).
