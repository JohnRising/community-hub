---
title: Running a node without Docker
lang: en-US
---


Here are the instructions if you want to build you own OP Mainnet, or OP Goerli, replica without relying on our images.
These instructions were generated on an Ubuntu 20.04 LTS box, but they should work with other systems too.

**Note:** This is *not* the recommended configuration.
While we did QA on these instructions and they work, the QA that the docker images undergo is much more extensive.

## Prerequisites

You’ll need the following software installed to follow this tutorial:

- [Git](https://git-scm.com/)
- [Go](https://go.dev/)
- [Node](https://nodejs.org/en/)
- [Pnpm](https://classic.yarnpkg.com/lang/en/docs/install/)
- [Foundry](https://github.com/foundry-rs/foundry#installation)
- [Make](https://linux.die.net/man/1/make)
- [jq](https://github.com/jqlang/jq)
- [direnv](https://direnv.net)
- [zstd](http://facebook.github.io/zstd/)

This tutorial was checked on:

| Software | Version    | Installation command(s) |
| -------- | ---------- | - |
| Ubuntu   | 20.04 LTS  | |
| git, curl, jq, make, and zstd | OS default | `sudo apt update` <br> `sudo apt install -y git curl make jq zstd`|
| Go       | 1.20       | `wget https://go.dev/dl/go1.20.linux-amd64.tar.gz` <br> `tar xvzf go1.20.linux-amd64.tar.gz` <br> `sudo cp go/bin/go /usr/bin/go` <br> `sudo mv go /usr/lib` <br> `echo export GOROOT=/usr/lib/go >> ~/.bashrc`
| Node     | 16.19.0    | `curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -` <br> `sudo apt-get install -y nodejs`
| pnpm     | 8.5.6      | `sudo npm install -g pnpm`
| yarn     | 1.22.19    | `sudo npm install -g yarn`
| Foundry  | 0.2.0      | `curl -L https://foundry.paradigm.xyz | bash` <br> `. ~/.bashrc` <br> `foundryup` 


## Build the Optimism Monorepo

1. Clone the [Optimism Monorepo](https://github.com/ethereum-optimism/optimism).

    ```bash
    cd ~
    git clone https://github.com/ethereum-optimism/optimism.git
    ```

1. Install required modules. 
   This is a slow process, while it is running you can already start building `op-geth`, as shown below.

    ```bash
    cd optimism
    pnpm install
    ```

1. Build the necessary packages inside of the Optimism Monorepo.

    ```bash
    make op-node
    pnpm build
    ```

## Build op-geth

1. Clone [`op-geth`](https://github.com/ethereum-optimism/op-geth):

    ```bash
    cd ~
    git clone https://github.com/ethereum-optimism/op-geth.git
    ```


1. Build `op-geth`:

    ```bash
    cd op-geth    
    make geth
    ```



## Get the data dir

The next step is to download the data directory for `op-geth`.

1. Download the correct data directory snapshot.

   - [OP Mainnet](https://datadirs.optimism.io/mainnet-bedrock.tar.zst)
   - [OP Goerli](https://datadirs.optimism.io/goerli-bedrock.tar.zst)

1. Create the data directory in `op-geth` and fill it.
   Note that these directions assume the data directory snapshot is at `~`, the home directory. Modify if needed.

   ```sh
   cd ~/op-geth
   mkdir datadir
   cd datadir
   tar xvf ~/*bedrock.tar
   ```

1. Create a shared secret with `op-node`:

   ```sh
   cd ~/op-geth
   openssl rand -hex 32 > jwt.txt
   cp jwt.txt ~/optimism/op-node
   ```

## Scripts to start the different components

### `op-geth`

This is the script for OP Goerli.
For OP Mainnet (or other OP networks in the future), [get the sequencer URL here](../../useful-tools/networks.md)).

```
#! /usr/bin/bash

SEQUENCER_URL=https://goerli-sequencer.optimism.io/

cd ~/op-geth

./build/bin/geth \
  --ws \
  --ws.port=8546 \
  --ws.addr=0.0.0.0 \
  --ws.origins="*" \
  --http \
  --http.port=8545 \
  --http.addr=0.0.0.0 \
  --http.vhosts="*" \
  --http.corsdomain="*" \
  --authrpc.addr=localhost \
  --authrpc.jwtsecret=./jwt.txt \
  --authrpc.port=8551 \
  --authrpc.vhosts="*" \
  --datadir=/data \
  --verbosity=3 \
  --rollup.sequencerhttp=$SEQUENCER_URL \
  --nodiscover \
  --syncmode=full \
  --maxpeers=0 \
  --datadir ./datadir \
  --snapshot=false
```


::: info Snapshots

For the initial synchronization it's a good idea to disable snapshots (`--snapshot=false`) to speed it up. 
Later, for regular usage, you can remove that option to improve geth database integrity.

:::

### `op-node`

- Change `<< URL to L1 >>` to a service provider's URL for the L1 network (either L1 Ethereum or Goerli).
- Set `L1KIND` to the network provider you are using (alchemy, infura, etc.).
- Set `NET` to either `goerli` or `mainnet`.


```
#! /usr/bin/bash

L1URL=  << URL to L1 >>
L1KIND=alchemy
NET=goerli

cd ~/optimism/op-node

./bin/op-node \
        --l1=$L1URL  \
        --l1.rpckind=$L1KIND \
        --l2=http://localhost:8551 \
        --l2.jwt-secret=./jwt.txt \
        --network=$NET \
        --rpc.addr=0.0.0.0 \
        --rpc.port=8547

```        




## The initial synchornization

The datadir provided by Optimism is not updated continuously, so before you can use the node you need a to synchronize it.

During that process you get log messages from `op-node`, and nothing else appears to happen.

```
INFO [06-26|13:31:20.389] Advancing bq origin                      origin=17171d..1bc69b:8300332 originBehind=false
```

That is normal - it means that `op-node` is looking for a location in the batch queue. 
After a few minutes it finds it, and then it can start synchronizing.

While it is synchronizing, you can expect log messages such as these from `op-node`:

```
INFO [06-26|14:00:59.460] Sync progress                            reason="processed safe block derived from L1" l2_finalized=ef93e6..e0f367:4067805 l2_safe=7fe3f6..900127:4068014 l2_unsafe=7fe3f6..900127:4068014 l2_time=1,673,564,096 l1_derived=6079cd..be4231:8301091
INFO [06-26|14:00:59.460] Found next batch                         epoch=8e8a03..11a6de:8301087 batch_epoch=8301087 batch_timestamp=1,673,564,098
INFO [06-26|14:00:59.461] generated attributes in payload queue    txs=1  timestamp=1,673,564,098
INFO [06-26|14:00:59.463] inserted block                           hash=e80dc4..72a759 number=4,068,015 state_root=660ced..043025 timestamp=1,673,564,098 parent=7fe3f6..900127 prev_randao=78e43d..36f07a fee_recipient=0x4200000000000000000000000000000000000011 txs=1  update_safe=true
```

And log messages such as these from `op-geth`:

```
INFO [06-26|14:02:12.974] Imported new potential chain segment     number=4,068,194 hash=a334a0..609a83 blocks=1         txs=1         mgas=0.000  elapsed=1.482ms     mgasps=0.000   age=5mo2w20h dirty=2.31MiB
INFO [06-26|14:02:12.976] Chain head was updated                   number=4,068,194 hash=a334a0..609a83 root=e80f5e..dd06f9 elapsed="188.373µs" age=5mo2w20h
INFO [06-26|14:02:12.982] Starting work on payload                 id=0x5542117d680dbd4e
```

### How long will the synchronization take?

To estimate how long the synchronization will take, you need to first find out how many blocks you synchronize in a minute. 

You can use this script, which uses [Foundry](https://book.getfoundry.sh/). 



```sh
#! /usr/bin/bash

export ETH_RPC_URL=http://localhost:8545
CHAIN_ID=`cast chain-id`
echo Chain ID: $CHAIN_ID
echo Please wait

if [ $CHAIN_ID -eq 10 ]; then
  L2_URL=https://mainnet.optimism.io
fi


if [ $CHAIN_ID -eq 420 ]; then
  L2_URL=https://goerli.optimism.io
fi


T0=`cast block-number` ; sleep 60 ; T1=`cast block-number`
PER_MIN=`expr $T1 - $T0`
echo Blocks per minute: $PER_MIN


if [ $PER_MIN -eq 0 ]; then
    echo Not synching
    exit;
fi

# During that minute the head of the chain progressed by thirty blocks
PROGRESS_PER_MIN=`expr $PER_MIN - 30`
echo Progress per minute: $PROGRESS_PER_MIN


# How many more blocks do we need?
HEAD=`cast block-number --rpc-url $L2_URL`
BEHIND=`expr $HEAD - $T1`
MINUTES=`expr $BEHIND / $PROGRESS_PER_MIN`
HOURS=`expr $MINUTES / 60`
echo Hours until sync completed: $HOURS

if [ $HOURS -gt 24 ] ; then
   DAYS=`expr $HOURS / 24`
   echo Days until sync complete: $DAYS
fi
```


## Operations

It is best to start `op-geth` first and shut it down last.