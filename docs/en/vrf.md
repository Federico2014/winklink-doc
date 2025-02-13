# WinkLink Random number Service

## Overview
Verifiable Random Function (VRF) is the public-key version of a keyed cryptographic hash, which can be used as random number. Only the holder of the private key can compute the hash, but anyone with public key can verify the correctness of the hash.
VRF can be used to generate secure and reliable random numbers.

Random number is determined by seed (provided by users), nonce (private state of VRFCoordinator contract) , block hash (the block of the request event) and private key of oracle node.

The generation process of VRF is:
- A Dapp contract sends out an on-chain request for random number;
- Once the off-chain oracle node listens for the request, it generates a random number attaching the cryptographic proof to make the generated random number verifiable, and then submits them back to an oracle contract (VRFCoordinator);
- Once the random number proof is verified by the oracle contract, the random number is published to the Dapp contract through a callback function.

The process above ensures that the random number cannot be tampered with nor manipulated by anyone, including oracle operators, miners, users and even smart contract developers.

WinkLink VRF is a provably-fair and verifiable source of randomness designed for Dapp contracts. Dapp contract developers can use WinkLink VRF as a tamper-proof RNG (Random Number Generator) to build reliable smart contracts for any applications which rely on unpredictable random number:
- Blockchain games and NFTs
- Random assignment of duties and resources (e.g. randomly assigning judges to cases)
- Choosing a representative sample for consensus mechanisms

This article describes how to deploy and use the WinkLink VRF service.

## Before you start

Maintainers for WinkLink need to understand how the TRON platform works, and know about smart contract deployment and the process of calling them. You're suggested to read related [TRON official documents](https://cn.developers.tron.network/), particularly those on contract deployment on TronIDE.

Prepare the node account. You should to read related [Node account preparation doc](https://doc.winklink.org/v1/doc/en/deploy.html#prepare-node-account).

## VRFCoordinator Contract
VRFCoordinator contract is deployed on the TRON public chain with the following features:

- Receive random number requests from Dapp contract and emit VRFRequest event
    - WIN transfer as fees, will be sent along with the request
- Accept random number and the proof submitted from WinkLink node
    - VRFCoordinator contract will verify the proof before sending the random number to Dapp contract
- Calculate the WinkLink node rewards for the request fulfilment

VRFCoordinator contract code is available at [VRFCoordinator.sol](https://github.com/wink-link/winklink/blob/feature/rename2wink/tvm-contracts/v1.0/VRF/VRFCoordinator.sol) .

WIN token address, WinkMid contract address and BlockHashStore contract address are needed in the constructor function when deploying a VRFCoordinator contract.

For convenience, Nile testnet has deployed `WinkMid` contract and encapsulated the `WIN` token on it. Developers can use this contract address directly without additional deployment. Users can also claim test TRX and WIN tokens from the Faucet address provided by Nile testnet.

::: tip Nile Testnet

- WIN TRC20 Contract Address: `TNDSHKGBmgRx9mDYA9CnxPx55nu672yQw2`
- WinkMid Contract Address: `TFbci8j8Ja3hMLPsupsuYcUMsgXniG1TWb`
- Testnet Faucet: <https://nileex.io/join/getJoinPage>
  :::

## Node Deployment
For node deployment, please refer to [WinkLink Node Deploy Doc](https://doc.winklink.org/v1/doc/en/deploy.html) .  This section only lists the differences of VRF node deployment.

WinkLink node should be deployed after the VRFCoordinator contract is deployed.

WinkLink node (project directory `node`) code is available at: <https://github.com/wink-link/winklink/tree/feature/rename2wink/node>.

After compilation, `node-v1.0.jar` will be stored in `node/build/libs/` under the project root directory.

### Node Configuration

After the node configuration file is confirmed, it is required to create a `vrfKeyStore.yml` file and set the private key for VRF (support multiple VRF private keys in one node):

```text
privateKeys:
  - *****(private key in hexadecimal format)
```

Support for dynamically updating vrfkeystore without restarting the node server. The steps are:

First, add a new VRF private key to the `vrfKeyStore.yml` 

Second, execute the following command:

```sh
curl --location --request GET 'http://localhost:8081/vrf/updateVRFKey/vrfKeyStore.yml'
```

::: tip
It is for important safety concern that you use file to provide private information instead of command line. In the production environment, set the permission of the private file `vrfKeyStore` as 600, meaning that only the owner can read or write the data.
:::

### Start a Node

All configuration files need to be copied to the directory where your node is running in, use the command `cp node/src/main/resource/*.yml ./`.
At the same time, the `tronApiKey` part of the `application dev` file needs to be filled with apikey.

Start your WinkLink node using the following command:

```sh
java -jar node/build/libs/node-v1.0.jar -k key.store -vrfK vrfKeyStore.yml
```

Configuration items can also be specified using a command line. For example:

mainnet:
```sh
java -jar node/build/libs/node-v1.0.jar --server.port=8081 --spring.profiles.active=dev --key key.store  --vrfKey vrfKeyStore.yml
```
nile testnet:
```sh
java -jar node/build/libs/node-v1.0.jar --env dev --server.port=8081 --spring.profiles.active=dev --key key.store  --vrfKey vrfKeyStore.yml
```

Determine whether your WinkLink node is running properly using the following command:

```sh
tail -f logs/tron.log
```

::: warning
Your node account must have enough TRX tokens for contract calls.
You can apply testnet tokens at Testnet Faucet.
:::

### Add a Job to Your Node

The job of your node represents the data service that your node supports, and each job has a unique 32-byte ID. 

When your WinkLink node is running properly, you can add a job to your node via HTTP API:

Example: (change the parameter below:  `address`  is the VRFCoordinator contract address deployed in the steps above; 
`publicKey` is the compressed value of the node's public key, which can be obtained by viewing the terminal display after the node starts, and the corresponding item is `eckey compressed`)

```sh
curl --location --request POST 'http://localhost:8081/job/specs' \
  --header 'Content-Type: application/json' \
    --data-raw '{
    "initiators": [
        {
        "type": "randomnesslog",
        "params": {
            "address": "TYmwSFuFuiDZCtYsRFKCNr25byeqHH7Esb"
        }
        }
    ],
    "tasks": [
        {
        "type": "random",
        "params": {
        "publicKey":"0x024e6bda4373bea59ec613b8721bcbb56222ab2ec10b18ba24ae369b7b74ab1452"
        }
        },
        {
        "type": "trontx",
        "params": {
            "type": "TronVRF"
        }
	}
    ]
    }'
```

### Query Jobs

Request example:

```sh
curl --location --request GET 'http://localhost:8081/job/specs'
```

## Authorize a Node Account

Node account needs authorization to submit data to VRFCoordinator contract, otherwise error will be reported.

The owner of the VRFCoordinator contract is required to call the contract below and add the node account to the whitelist:

```js
  function registerProvingKey(uint256 _fee, address _oracle, bytes calldata _publicProvingKey, bytes32 _jobID)
```

`_fee` is the minimum WIN token cost required for the registration node to generate random numbers, 
`_oracle` is the address of the registered node, which is used to receive the WIN token paid by DAPP ,
`_publicProvingKey` is the public key used by the registration node to generate random numbers, which is `x || y`, 
`_jobID` is the jobID of VRF service of the node.

Call example: `registerProvingKey（10,TYmwSFuFuiDZCtYsRFKCNr25byeqHH7Esb,
0x4e6bda4373bea59ec613b8721bcbb56222ab2ec10b18ba24ae369b7b74ab145224d509bc2778e6d1c8a093522ba7f9b6669a9aef57d2231f856e4b594ad5f4ac,
04d773890bc347f88544dc85bea24985）`

## Dapp Contract

Contract code is available at  [VRFD20.sol](https://github.com/wink-link/winklink/blob/feature/rename2wink/tvm-contracts/v1.0/VRF/VRFD20.sol)

### Dapp Contract Deployment

Some parameters are needed in the constructor function when deploying an Dapp contract

```js
  constructor(address vrfCoordinator, address win, address winkMid, bytes32 keyHash, uint256 fee)
```
`vrfCoordinator` the address of VRFCoordinator, `win` WIN token address, `winkMid` WinkMid contract address,
`keyHash` the hash value of the public key of the registered node, which can be obtained by calling the `hashofkeybytes` function of the VRFCoordinator contract (input is `x || y`).
`fee` the WIN token fee payed for generating random number, and its value should be greater than the fee required by random number node.

Example:  `constructor（TUeVYd9ZYeKh87aDA9Tp7F5Ljc47JKC37x,TNDSHKGBmgRx9mDYA9CnxPx55nu672yQw2,
TFbci8j8Ja3hMLPsupsuYcUMsgXniG1TWb,0xe4f280f6d621db4bccd8568197e3c84e3f402c963264369a098bb2f0922cb125,12）`.

### Transfer WIN Tokens to the Contract

VRFD20 contract needs to call the VRFCoordinator contract, so there should be enough WIN tokens in the contract account. You can transfer a certain amount of WIN tokens for the contract through the transfer service or the TestNet Faucet.

### Call the Dapp Contract

Use the following interface to request random number:

```js
function rollDice(uint256 userProvidedSeed, address roller)
```

`userProvidedSeed` the seed provided by the user,`roller` You can fill in any address at present.
Call example: `rollDice(0x852f725894485e4979af5ea47ddd90cc68ea1ac0f4b99e52e9b91fa35a7204e2, TL44GNkjETr2JumQHgYJF842oyE6h2inoR)`.
