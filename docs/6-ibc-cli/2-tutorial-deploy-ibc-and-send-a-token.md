---
title: "Deploy IBC and send a token"
description: "Bring up two chains, deploy IBC on both, and move an Interchain Fungible Token from one to the other."
docci:
  # CI only. The reader runs the commands exactly as they are printed; these
  # substitutions keep the run off the reader's real ~/.ibc, make the clone
  # re-runnable, and move the relayer off a port that is already taken here.
  replace-text:
    # The reader clones the published repository. CI clones the checkout it is
    # running in, so the page tests the code it ships beside rather than
    # whatever is on the default branch at the time. Without this a change to
    # the CLI and the page that documents it can never be tested together.
    #
    # The smudge filter is skipped: the spec images are LFS objects a local
    # clone cannot resolve, and the tutorial reads none of them.
    #
    # GITHUB_WORKSPACE falls back to the working directory, so the page can be
    # run by hand on a Linux box without setting anything.
    - "git clone https://github.com/cosmos/ibc.git &&;{ [ -d ibc ] || GIT_LFS_SKIP_SMUDGE=1 git clone --depth 1 \"file://${GITHUB_WORKSPACE:-$PWD}\" ibc; } &&"
    - "./bin/ibc ;./bin/ibc --home ${DOCCI_IBC_HOME:-$PWD/.docci-ibc} "
    # The transfer's hash is captured through a pipe, so the exit status is
    # jq's and a failed send reads as success with an empty hash. The failure
    # then surfaces twenty retries later in the status step, pointing at the
    # wrong command. pipefail makes the send fail where it fails.
    - "TX=$(;TX=$(set -o pipefail; "
    # ...and docci runs blocks with `set -e`, which bash does not apply to the
    # left side of `&&`. With the echo chained on, a failed assignment does not
    # abort the block, execution reaches docci's trailing `trap - DEBUG`, and
    # that exits 0. Unchain the echo so the assignment stands on its own.
    - " && echo \"$TX\"; ; echo \"$TX\""
    - "./bin/ibc --home ${DOCCI_IBC_HOME:-$PWD/.docci-ibc} config new;./bin/ibc --home ${DOCCI_IBC_HOME:-$PWD/.docci-ibc} config new && sed -i'' -e 's|listenAddr: 0.0.0.0:3000|listenAddr: 0.0.0.0:3100|' ${DOCCI_IBC_HOME:-$PWD/.docci-ibc}/ibc.yml"
  cleanup:
    - "cd ibc/examples/besu-to-besu && ./setup.sh clean || true"
    - "rm -rf ${DOCCI_IBC_HOME:-$PWD/.docci-ibc}"
---

This tutorial walks you through all the steps to deploy and run IBC on two EVM chains using IBC CLI.

By the end, you'll have the following:

- IBC CLI installed and configured
- IBC contracts deployed on two local Besu chains
- IBC connections established between the chains
- A relayer running
- Attestors running
- An Interchain Fungible Token that can be transferred between the chains

## Prerequisites

- [Docker](https://docs.docker.com/get-started/get-docker/) installed and running
- [Go](https://go.dev/doc/install) v1.26.6 or later
- [jq](https://jqlang.org/download/) installed
- [Git](https://git-scm.com/downloads) installed

Ports `8545`, `8546`, `8745`, `8746`, `9545` and `9745` must be free for the two chains, and `3000` for the relayer.

## 1. Set up your environment

Start by installing IBC CLI and running two local Besu chains.

1. Clone the IBC repository and build the binary:

<!-- docci name="build the ibc binary" -->

```bash
git clone https://github.com/cosmos/ibc.git && cd ibc/cli && make build
```

The binary is located at `bin/ibc`.

2. This tutorial uses two local [Besu](https://docs.besu-eth.org/) chains, which are EVM-based. Start both chains:

<!-- docci name="start both chains" delay-after=2 -->

```bash
../examples/besu-to-besu/setup.sh
```

The script initializes two single-validator Besu chains, starts them, and waits until both answer RPC calls. It finishes by printing something similar to the following:

<!-- docci expect-output -->

```
[<...>] Chain status:
[<...>]   besu-a (chain-id 41001) RPC=http://localhost:8545 WS=ws://localhost:8546 block=<...>
[<...>]   besu-b (chain-id 41002) RPC=http://localhost:8745 WS=ws://localhost:8746 block=<...>
[<...>] Chains are live and producing blocks.
```

Each line is prefixed with the time it was printed, and the block number is whatever height the chains have reached by then. If you have run the script before without `./setup.sh clean`, the chains resume from their previous height rather than from block 1.

Now there are two live chains, each producing blocks.

3. You can use the following command to look at the funded accounts included in the setup script:

<!-- docci name="list funded accounts" output-contains="0x" -->

```bash
../examples/besu-to-besu/setup.sh accounts
```

Five accounts are derived from one mnemonic and funded on both chains, so the same key works on either side. The next step uses these.

## 2. Create a config and keys

IBC CLI uses one config file for all its operations.

1. Start by creating the config file:

<!-- docci name="create the config" -->

```bash
./bin/ibc config new
```

The config is stored in `~/.ibc/ibc.yml` by default.

This is the home directory for everything that follows: the config, the keystore, the deployment records, and the relayer's database.

2. Import a deployer key from the accounts created during the chain setup. This key becomes the access manager's admin, which governs the IBC router and the GMP app. It is also the default owner of any token you deploy.

Passing `--populate-config` adds each key to the config as a named signer. Later commands then name a key by alias instead of by path.

<!-- docci name="import the deployer key" -->

```bash
./bin/ibc keys add ecdsa deployer --private-key 0x33fa40f84e854b941c2b0436dd4a256e1df1cb41b9c1c0ccc8446408c19b8bf9 --populate-config
```

<!-- docci expect-output -->

```json
{
  "evmAddress": "0x58A57ed9d8d624cBD12e2C467D34787555bB1b25",
  "path": "<...>/keys/deployer.json",
  "publicKey": "0x03a70d1ef368ad99e90d509496e9888ee7404e4f4d360376bf521d769cf0c4de46",
  "type": "ecdsa"
}
```

The address and public key are derived from the private key above, so they are the same every time. The path is wherever your IBC home is.

3. Import a relayer key, which signs packet delivery and pays for the relayer's gas:

<!-- docci name="import the relayer key" output-contains="evmAddress" -->

```bash
./bin/ibc keys import ecdsa relayer --private-key 0xd46507376b3a8a4af4f1f934375df25213a09857a3ed1086ba284c82d387904b --populate-config
```

4. Generate an attestor key for each chain. These sign attestations and never need funding. Never reuse an attestor key across clients, so each chain gets its own: see [Attestors](../4-light-clients/2-attestors.md#attestor-keys).

<!-- docci name="create the attestor key for 41001" output-contains="evmAddress" -->

```bash
./bin/ibc keys new ecdsa attestor-41001 --populate-config
```

<!-- docci name="create the attestor key for 41002" output-contains="evmAddress" -->

```bash
./bin/ibc keys new ecdsa attestor-41002 --populate-config
```

5. Next, register the first chain's details in the config. This passes the chain ID, RPC and websocket endpoints, and deployer key.

<!-- docci name="register chain 41001" -->

```bash
./bin/ibc config add-chain --chain-id 41001 --rpc http://localhost:8545 --ws ws://localhost:8546 --deployer deployer
```

6. Register the second chain's details:

<!-- docci name="register chain 41002" -->

```bash
./bin/ibc config add-chain --chain-id 41002 --rpc http://localhost:8745 --ws ws://localhost:8746 --deployer deployer
```

The `--deployer` flag names the key that signs deployments on that chain, so you won't need to pass it again.

## 3. Deploy IBC on both chains

With both chains registered, deploy the IBC contracts on each.

1. Deploy the IBC core on each chain. This deploys the ICS26 router and the access manager:

<!-- docci name="deploy core on 41001" output-contains="core stack on chain 41001" -->

```bash
./bin/ibc deploy core --chain 41001 --yes
```

<!-- docci name="deploy core on 41002" output-contains="core stack on chain 41002" -->

```bash
./bin/ibc deploy core --chain 41002 --yes
```

Each run sends four transactions: an access manager, the router implementation, the router behind a proxy, and one call that opens the packet-delivery methods to any caller. Your deployer key is the access manager's admin. It logs each transaction as it is mined and finishes with the step marked `executed`.

2. Deploy an attestation light client on each chain. Each deployment tracks the state of the other chain:

<!-- docci name="deploy the client on 41001" output-contains="executed" -->

```bash
./bin/ibc deploy client attestation --chain 41001 --counterparty-chain 41002 --attestors attestor-41002 --threshold 1 --yes
```

<!-- docci name="deploy the client on 41002" output-contains="executed" -->

```bash
./bin/ibc deploy client attestation --chain 41002 --counterparty-chain 41001 --attestors attestor-41001 --threshold 1 --yes
```

The `--attestors` flag takes the aliases from step 4. Each resolves to that key's address, and those addresses become the client's attestation set on chain.

The `--threshold` flag is how many of them must sign for the client to accept a proof. Both are fixed on chain when the client is deployed.

Note the crossed pairing. A client verifies the state of the chain it tracks, so the client on 41001 tracks 41002 and trusts the attestor watching 41002.

Both sides share one client identifier, derived from the two chain IDs in sorted order, so it reads the same on either chain. Here it is `cli-41001-41002`, which will be used later to identify the connection.

## 4. Deploy the application

The next steps deploy the General Message Passing (GMP) app and the Interchain Fungible Token (IFT) contracts on each chain. IFT uses GMP to send and receive messages between the chains and facilitate token transfers.

1. Deploy the GMP app on each chain. This allows cross-chain contract calls between the chains:

<!-- docci name="deploy gmp on 41001" output-contains="executed" -->

```bash
./bin/ibc deploy gmp --chain 41001 --yes
```

<!-- docci name="deploy gmp on 41002" output-contains="executed" -->

```bash
./bin/ibc deploy gmp --chain 41002 --yes
```

2. Deploy an IFT contract on each chain. This is the token that will be transferred between the chains:

<!-- docci name="deploy the token on 41001" output-contains="ift token deployed" -->

```bash
./bin/ibc deploy ift --name "Demo Token" --symbol DEMO --chain 41001 --yes
```

<!-- docci name="deploy the token on 41002" output-contains="ift token deployed" -->

```bash
./bin/ibc deploy ift --name "Demo Token" --symbol DEMO --chain 41002 --yes
```

Each deployment logs the address it landed at, which differs every run.

3. Use the following commands to read both token addresses out of the deployment records. These will be used in later steps:

<!-- docci name="read the token addresses" -->

```bash
IFT_A=$(./bin/ibc deploy show 41001 | jq -r '.tokens[0].address')
IFT_B=$(./bin/ibc deploy show 41002 | jq -r '.tokens[0].address')
```

If you need to view these addresses later, the `deploy show` command prints a chain's deployment record as JSON.

4. The next step is to link the two token contracts together. Registering a bridge tells each token to accept mints from its counterpart on the other chain. A transfer then burns tokens on the source chain and mints them on the destination.

<!-- docci name="register the token bridge" output-contains="executed" -->

```bash
./bin/ibc deploy ift-bridge --chain-a 41001 --ift-a "$IFT_A" --chain-b 41002 --ift-b "$IFT_B" --yes
```

This command registers both sides. It ties each token to the client pointing at the other chain, which is how a transfer knows where to land.

## 5. Configure and start the relayer

Next, you'll need to configure the relayer to start sending packets between the two chains.

1. Generate and save the relayer's configuration:

<!-- docci name="render the relayer config" -->

```bash
./bin/ibc deploy render-config 41001 41002 \
  --signer-a relayer --signer-b relayer \
  --populate-config --yes
```

This merges the deployed router addresses, connection, and local attestors into your existing config, preserving settings such as `server`, `db`, and `signers`. It prints the complete config and writes it to `~/.ibc/ibc.yml`. Without `--yes` it stops and asks you to confirm first.

The two signer flags select the key that submits relay transactions on each chain. Set both for this new connection; each is checked against your configured signers. You imported `relayer` in step 3 of section 2.

The attestors section declares both of your attestor keys as `type: local`. This means the relayer will run the attestors in the same process.

2. Use the validate command to check the result against both chains before starting anything:

<!-- docci name="validate the config against both chains" -->

```bash
./bin/ibc config validate --live
```

<!-- docci expect-output -->

```json
{
  "path": "<...>/ibc.yml",
  "status": "valid"
}
```

If the status is anything other than `valid`, the error names the chain or signer that failed; fix that before starting the relayer.

3. Now you'll need to open a new terminal to start the relayer and attestors. Leave your first terminal open. You'll come back to it in the next step.

<!-- docci name="start the relayer" background -->

```bash
# open a new terminal and start the relayer
./bin/ibc relayer run
```

It prints a readiness line once both chains are connected:

<!-- docci expect-output -->

```
level=INFO msg=Readiness module=bootstrap readiness="{Event:ready ChainsConnected:[41001 41002] HTTP:[::]:<...>}"
```

The readiness line is how you know it is up. "Dual mode" means the attestors are running inside this process, because the rendered configuration declared them local.

Leave it running, and go back to your first terminal for the rest of the tutorial.

## 6. Transfer a token

The next steps will mint the token and send it to the other chain.

1. Mint an initial supply of the token into the deployer account.

<!-- docci name="mint the initial supply" output-contains="txHash" -->

```bash
./bin/ibc tx ift mint --chain 41001 --ift "$IFT_A" --to deployer --from deployer --amount 100000000000000000000
```

The `--from` key must be the token's owner, which is the deployer. Amounts are in base units, so this is 100 tokens at 18 decimals.

2. Send 10 tokens from one chain to the other, keeping the transaction hash. `$IFT_A` is the token address you stored earlier.

> **Note:** The `--timeout` flag sets how long the packet has to be delivered, counted from when you send. It defaults to 15 minutes and cannot exceed one day. A packet that misses that window can only be timed out and refunded, and the deadline cannot be changed after the send.

<!-- docci name="send the transfer" -->

```bash
TX=$(./bin/ibc tx ift send --chain 41001 --ift "$IFT_A" --client-id cli-41001-41002 --to deployer --from deployer --amount 10000000000000000000 | jq -r '.txHash') && echo "$TX"
```

The send prints `{"txHash": "0x..."}`. That hash is stored in `$TX`, which the next command uses:

<!-- docci expect-output -->

```
0x<...>
```

Once this transaction is mined, the tokens are burned on the source chain. They are minted on the destination only when the packet is delivered. If delivery fails, or the packet times out, the tokens are minted back to the sender once a relayer carries that outcome home.

3. The relayer will automatically detect that the IBC packet was created and relay it.

4. Now you can check the status of the transfer:

<!-- docci
  name="transfer reaches a terminal state"
  retry=20
  delay-per-cmd=3
-->

```bash
./bin/ibc relayer packets --chain-id 41001 --tx-hash "$TX"
```

It will read `PACKET_STATE_PENDING` for up to a minute, then it should read `PACKET_STATE_SUCCEEDED`:

<!-- docci expect-output -->

```json
{
  "packets": [
    {
      "state": "PACKET_STATE_SUCCEEDED",
      "sequenceNumber": "1",
      "sourceClientId": "cli-41001-41002",
      "sendTx": {
        "txHash": "0x<...>",
        "chainId": "41001"
      },
      "recvTx": {
        "txHash": "0x<...>",
        "chainId": "41002"
      },
      "ackTx": {
        "txHash": "0x<...>",
        "chainId": "41001"
      },
      "timeoutTx": null
    }
  ],
  "hasMore": false,
  "nextCursor": ""
}
```

This shows three transactions across two chains: the send you made, the receive on 41002, and the acknowledgement carried back to 41001. If it is still pending after a minute, check the terminal running the relayer for errors.

5. Confirm the tokens arrived on the destination chain:

<!-- docci name="destination balance" -->

```bash
./bin/ibc query ift balance --chain 41002 --ift "$IFT_B" --address deployer
```

<!-- docci expect-output -->

```json
{
  "address": "0x58A57ed9d8d624cBD12e2C467D34787555bB1b25",
  "balance": "10000000000000000000",
  "symbol": "DEMO"
}
```

Then check that the source chain was debited:

<!-- docci name="source balance" -->

```bash
./bin/ibc query ift balance --chain 41001 --ift "$IFT_A" --address deployer
```

<!-- docci expect-output -->

```json
{
  "address": "0x58A57ed9d8d624cBD12e2C467D34787555bB1b25",
  "balance": "90000000000000000000",
  "symbol": "DEMO"
}
```

Congratulations! You've deployed IBC and transferred a token between two chains.

## 7. Shut everything down

Stop the relayer with `Ctrl-C` in the terminal running it, then remove the two chains and their data:

<!-- docci name="tear down the chains" -->

```bash
cd ../examples/besu-to-besu && ./setup.sh clean && cd -
```

This stops both containers and deletes their volumes, so the next run starts from block 1. Your config, keys and deployment records stay in `~/.ibc`; delete that directory too if you want a completely clean slate.

## Next steps

- [Overview](1-overview.md) explains what the three parts are made of and how they fit together.
- [Run a standalone attestor](3-run-a-standalone-attestor.md) moves an attestor out of the relayer's process and into its own.
- [Run a standalone relayer](4-run-a-standalone-relayer.md) brings up a relayer against a deployment, including one that replaces a relayer that stopped.
