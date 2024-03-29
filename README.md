# Switchboard Function Example

<div align="center">
  <img src="https://github.com/switchboard-xyz/sbv2-core/raw/main/website/static/img/icons/switchboard/avatar.png" />
  <h1>Switchboard<br>Exchange Aggregator Function Example</h1>
  <p>
    <a href="https://discord.gg/switchboardxyz">
      <img alt="Discord" src="https://img.shields.io/discord/841525135311634443?color=blueviolet&logo=discord&logoColor=white" />
    </a>
    <a href="https://twitter.com/switchboardxyz">
      <img alt="Twitter" src="https://img.shields.io/twitter/follow/switchboardxyz?label=Follow+Switchboard" />
    </a>
  </p>
</div>

## Table of Content

- [Prerequisites](#prerequisites)
  - [Installing Docker](#installing-docker)
  - [Docker Setup](#docker-setup)
- [Components](#components)
  - [Contract](#contract)
  - [Switchboard Function](#switchboard-function)
  - [Publishing and Initialization](#publishing-and-initialization)
  - [Adding Funding to Function](#adding-funding-to-function)
  - [Printing Function Data](#printing-function-data)
- [Writing Switchboard Rust Functions](#writing-switchboard-rust-functions)
  - [Setup](#setup)
  - [Minimal Example](#minimal-switchboard-function)
  - [Testing your function](#testing-your-function)
  - [Deploying and maintenance](#deploying-and-maintenance)
- [Writing Receiver Contracts](#writing-receiver-contracts)
  - [Receiver Example](#receiver-example)

## Prerequisites

Before you can build and run the project, you'll need to have Docker installed on your system. Docker allows you to package and distribute applications as lightweight containers, making it easy to manage dependencies and ensure consistent behavior across different environments. Switchboard Functions are built and run within containers, so you'll need a docker daemon running to publish a new function.

### Installing Docker

If you don't have Docker installed, you can follow these steps to get it up and running:

1. **Linux**: Depending on your Linux distribution, you might need to use different package managers. For Ubuntu, you can use `apt`:

   ```bash
   sudo apt update
   sudo apt install docker.io
   ```

   For other distributions, consult your package manager's documentation.

2. **macOS**: You can install Docker Desktop for macOS by downloading the installer from the [Docker website](https://www.docker.com/products/docker-desktop) and following the installation instructions.

3. **Windows**: Similarly, you can install Docker Desktop for Windows from the [Docker website](https://www.docker.com/products/docker-desktop) and follow the provided instructions.

### Docker Setup

After installing Docker, make sure it's running by opening a terminal/command prompt and running:

```bash
docker --version
```

This should display the installed Docker version, confirming that Docker is installed and running properly.

You'll need to login to docker. If you don't yet have an account, you'll need one to publish images to dockerhub. You can sign up at [https://hub.docker.com](https://hub.docker.com).

```bash
docker login --username <your-username> --password <your-password>
```

### Install script dependencies

```bash
pnpm i
```

### Install Switchboard cli

```bash
pnpm i -g @switchboard-xyz/cli@latest
sb evm function --help
```

## Components

### Contract

This example contract acts as the ingester of the switchboard-function in this directory to fetch implied volatility parameters via deribit. The example contract is an example of the [ERC2535 diamond contract pattern](https://autifynetwork.com/exploring-erc-2535-the-diamond-standard-for-smart-contracts/) so it can be extended and upgraded for your needs.

When you deploy this contract, it will await to be bound to a switchboard function calling into it.

#### Picking a network and setting up your environment

- set the `SWITCHBOARD_ADDRESS` env variable to target whichever address is appropriate for the network you're targeting
- for arbitrum sepolia testnet, this is: `0x0d251E9F64Fb3a146af61bB99d80471893b20cCF`

To first deploy the contract, run:

```bash
# ex:
# pnpm deploy:coredaotestnet
# pnpm deploy:coredaomain
# pnpm deploy:arbitrumtestnet
pnpm deploy:${NETWORK_NAME}
```

More deploy commands are available in [package.json](./package.json) scripts.

You will see the last line of this script output

```bash
export RECEIVER_ADDRESS=<RECEIVER_ADDRESS>
```

### Switchboard Function

Export the address to your environment and navigate to `./switchboard-function/`

The bulk of the function logic can be found in [./switchboard-function/src/main.rs](switchboard-function/src/main.rs).

Build functions from the `switchboard-function/` directory with

```bash
cd switchboard-function
make build
```

### Publishing and Initialization

You'll also need to pick a container name that your switchboard function will use on dockerhub.

```bash
export CONTAINER_NAME=your_docker_username/switchboard-function
export RECEIVER_ADDRESS=<RECEIVER_ADDRESS>
```

Here, set the name of your container and deploy it using:

```bash
cd switchboard-function
export CONTAINER_NAME=your_docker_username/switchboard-function
export RECEIVER_ADDRESS=<RECEIVER_ADDRESS>
make publish
```

Common gotcha: make sure this published container is publically visible

After this is published, you are free to make your function account to set the rate of run for the function.

You'll also need to create a file with your private key in it. This is used to sign the transactions that will be sent to the switchboard contract.

### Initializing the function

You can use the Switchboard cli to bind this docker container to an on-chain representation:

```bash
export SWITCHBOARD_ADDRESS_ARBITRUM_TESTNET=0x0d251E9F64Fb3a146af61bB99d80471893b20cCF
export QUEUE_ID=0x3e84bb41e96F90A93D0Ce930e75Cf47a2b262Ace # default testnet queue
export MEASUREMENT=<YOUR CONTAINER MEASUREMENT>
sb evm function create $QUEUE_ID --container ${CONTAINER_NAME} --containerRegistry dockerhub  --mrEnclave ${MEASUREMENT?} --name "options_example"  --chain arbitrum --account /path/to/signer --network testnet --programId $SWITCHBOARD_ADDRESS_ARBITRUM_TESTNET
...
export FUNCTION_ID=<YOUR FUNCTION ID>
sb evm routine create $FUNCTION_ID --chain $CHAIN --schedule "*/10 * * * * *" --account /path/to/keypair --network $CLUSTER --programId $SWITCHBOARD_ADDRESS_ARBITRUM_TESTNET --params="YOUR_CALL_PARAMS"
...
export ROUTINE_ID=<YOUR ROUTINE ID>
sb evm routine fund ${ROUTINE_ID?} --chain $CHAIN --fundAmount 0.01 --account /path/to/keypair --network $CLUSTER --programId $SWITCHBOARD_ADDRESS_ARBITRUM_TESTNET
```

### Printing the state of your callback

This repo contains an example script to view the current verified deribit implied volatility info
currently in contract:

```bash
npx hardhat run --network arbitrumTestnet scripts/get_state.ts
```

## Writing Switchboard Rust Functions

In order to write a successfully running switchboard function, you'll need to import `switchboard-evm` to use the libraries which communicate the function results (which includes transactions to run) to the Switchboard Verifiers that execute these metatransactions.

Update the switchboard-function directory to suit your target program's needs. The example function in this directory is a simple example of how to use the switchboard-evm library to communicate with the switchboard network.

### Testing your function

### NOTE: LOCAL TESTING WILL ONLY WORK ON LINUX MACHINES

We can't guarantee that the function will run on the blockchain, but we can test that it compiles and runs locally.

Run the following to test your function:

```bash
sb evm function test --chain arbitrum --parameters "" --network testnet
```

Successful output:

```bash
WARNING: Error generating quote. Function will not be able to be transmitted correctly.
FN_OUT: 7b2276657273696f6e223a312c2271756f7465223a5b5d2c22666e5f6b6579223a5b3134342c32332c3233322c34342c39382c32302c39372c3232392c3138392c33302c3235322c3133362c37362c332c3136382c3130362c3138322c34352c3137352c3137325d2c227369676e6572223a5b3135382c32332c3137302c3133322c3230302c3130322c35302c38352c31302c3134382c3235322c35372c3132362c372c31372c32352c37322c3131342c38322c3134365d2c22666e5f726571756573745f6b6579223a5b5d2c22666e5f726571756573745f68617368223a5b5d2c22636861696e5f726573756c745f696e666f223a7b2245766d223a7b22747873223a5b7b2265787069726174696f6e5f74696d655f7365636f6e6473223a313639313633383836332c226761735f6c696d6974223a2235353030303030222c2276616c7565223a2230222c22746f223a5b38332c3130372c3135352c35382c39382c3132382c37332c3233392c3134382c3133332c3133342c33392c3131382c31362c34382c3235302c3130372c3133382c3234382c3135375d2c2266726f6d223a5b3135382c32332c3137302c3133322c3230302c3130322c35302c38352c31302c3134382c3235322c35372c3132362c372c31372c32352c37322c3131342c38322c3134365d2c2264617461223a5b3136302c3232332c3131392c3130362...
```

The `error` above simply means your function could not produce a secure signature for the test since it was run outside the enclave.

### Deploying and Maintenance

After you publish the function and create it on the blockchain, you must keep the function escrow account funded to cover gas fees. Revisions to the function can be made by deploying a new version and updating the function config on-chain.

## Writing Receiver Contracts

While Switchboard Functions can call back into any number of on-chain functions, it's useful to limit access to some privileged functions to just _your_ Switchboard Function.

In order to do this you'll need to know the switchboard address you're using, and which functionId will be calling into the function in question.

Update the `src` directory for a look at an example receiver contract. This contract is deployed by the Switchboard, and is responsible for receiving data from the off-chain Switchboard Function, and making it available to other contracts.
