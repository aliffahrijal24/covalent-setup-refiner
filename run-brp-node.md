# Run BRP node

Docschevron\_right

Run BRP node

\
As we discussed in previous section, We can setup the nodes two ways so here in this section, you can follow steps accordingly to run your desired type of node:

* [Run with Docker Compose](https://www.covalenthq.com/docs/covalent-network/operator-onboarding-refiner/#run-with-docker-compose/) (Recommended method - Beginner)
* [Build & Run from Source](https://www.covalenthq.com/docs/covalent-network/operator-onboarding-refiner/#build--run-from-source/) (Optional method - Advanced)

#### &#x20;**Steps to Run Node with Docker:**

#### Clone Repo

Clone the [covalenthq/refiner](https://github.com/covalenthq/refiner/) repo.

```
git clone https://github.com/covalenthq/refiner
cd refiner
cat docker-compose-mbeam.yml
```

#### Check global environment variables

```
$ cat .envrc

export IPFS_PINNER_URL="http://127.0.0.1:3001"
export EVM_SERVER_URL="http://127.0.0.1:3002"

[[ -f .envrc.local ]] && source_env .envrc.local
```

#### Set local environment variables

```
$ touch .envrc.local

export BLOCK_RESULT_OPERATOR_PRIVATE_KEY="<<BRP-OPERATOR-PK-WITHOUT-0x-PREFIX>>"
export NODE_ETHEREUM_MAINNET="<<HTTPS-MOONBEAM-RPC-URL>>"
export IPFS_PINNER_URL="http://ipfs-pinner:3001"
export EVM_SERVER_URL="http://evm-server:3002"
export W3_AGENT_KEY="<<AGENT_KEY>>"
export DELEGATION_PROOF_FILE_PATH="<<DELEGATION_PROOF_FILE_PATH>>"
```

**Note**: `.envrc.local` overrides any env vars set with `.envrc` on calling `direnv allow .`

**Note**: When passing the private key into the env vars as above please remove the `0x` prefix so the private key env var has exactly 64 characters.

* `BLOCK_RESULT_OPERATOR_PRIVATE_KEY`: Your personal Block Result Producer (BRP) operator private key
* `AGENT_KEY & PROOF.OUT File`: Your personal web3.storage Agent key & Delegation file use by the `ipfs-pinner` service
* `IPFS_PINNER_URL`: Service (`ipfs-pinner`) used by rudder to access IPFS assets like Block Specimens (service is automatically invoked and run with the in repo docker compose files eg: `docker-compose-mbeam.yml`)
* `EVM_SERVER_URL`: Service (`evm-server`) used by rudder for stateless execution of Block Specimens into indexable (queryable) Block Results

#### Start Services

#### Load env vars into the shell.

```
$ direnv allow .

#make sure you see these being loaded
direnv: loading ~/rudder/.envrc
direnv: loading ~/rudder/.envrc.local
direnv: export +BLOCK_RESULT_OPERATOR_PRIVATE_KEY +ERIGON_NODE +EVM_SERVER_URL +IPFS_PINNER_URL +NODE_ETHEREUM_MAINNET +W3_AGENT_KEY
```

Copy over the delegation proof file to \~/.ipfs repo. You should have gotten this from Covalent in addition to the `W3_AGENT_KEY` value.

```
mv path_to_delegation_file ~/.ipfs/proof.out
```

#### Start all 3 services in the background for Moonbeam

```
$ docker compose -f "docker-compose-mbeam.yml" up -d --remove-orphans


[+] Running 3/3
 ⠿ Container rudder       Started                            3.2s
 ⠿ Container ipfs-pinner  Started                            1.9s
 ⠿ Container evm-server   Started                            1.8s
```

**NOTE**: On a system where an `ipfs-pinner` instance is already running, check the instruction in the [Appendix](https://www.covalenthq.com/docs/covalent-network/operator-onboarding-refiner/#appendix/) to run `refiner` docker alongside.

Monitor the logs for Block Result submissions.

```
$ docker compose -f "docker-compose-mbeam.yml" logs -f –tail 2

rudder       | [info] curr_block: 4591264 and latest_block_num:4591263
ipfs-pinner  | 2023/06/22 13:45:49 Received /health request: source= 127.0.0.1:54420 status= OK
rudder       | [info] curr_block: 4591264 and latest_block_num:4591263
ipfs-pinner  | 2023/06/22 13:46:00 Received /health request: source= 127.0.0.1:54430 status= OK
rudder       | [info] curr_block: 4591264 and latest_block_num:4591264
rudder       | [info] listening for events at 4591264
rudder       | [info] found 0 bsps to process
```

Check step 7 in [Run BRP](https://www.covalenthq.com/docs/covalent-network/operator-onboarding-refiner/#run-rudder/) from source section below for a successful Refiner stack run log output with performance metrics.

#### Deployment As Service Unit

Here you can find an example of a `systemd` service unit file that can be used to auto-start/restart of the docker-compose service for Refiner.

```
[Unit]
Description=Refiner docker compose
PartOf=docker.service
After=docker.service

[Service]
User=blockchain
Group=blockchain
Environment=HOME=/home/blockchain/tmp
Environment="BLOCK_RESULT_OPERATOR_PRIVATE_KEY=<<BRP-OPERATOR-PK-WITHOUT-0x-PREFIX>>"
Environment="NODE_ETHEREUM_MAINNET=<<HTTPS-MOONBEAM-RPC-URL>>"
Environment="IPFS_PINNER_URL=http://ipfs-pinner:3001" #service in docker
Environment="EVM_SERVER_URL=http://evm-server:3002" #service in docker
Environment="W3_AGENT_KEY=<<W3_AGENT_KEY>>"
Type=simple
ExecStart=docker compose -f "/home/blockchain/tmp/docker-compose-mbeam.yml" up --remove-orphans
Restart=always
TimeoutStopSec=infinity

[Install]
WantedBy=multi-user.target
```

After adding the env vars in their respective fields in the service unit file, enable the service and start it.

```
sudo systemctl enable rudder-compose.service
sudo systemctl start rudder-compose.service
```

**Note 1**: When passing the private key into the env vars as above please remove the `0x` prefix so the private key env var has exactly 64 characters.

**Note 2**: In order to run docker compose as a non-root user for the above shown service unit you need to create a docker group (if it doesn't exist) and add the user (say) "blockchain" to the docker group

```
sudo groupadd docker
sudo usermod -aG docker blockchain
sudo su - blockchain
docker run hello-world
```

**Hurray!** You have Successfully implemented the BRP Node using Docker Compose\


#### **Steps to Run Node with Source:** 

Here, to run the node from source we need to manually run all the three services step by step, follow this guide for implementing

* EVM-Server
* IPFS-Pinner
* Block Results Producer

#### Run EVM-Server

The EVM-Server is a stateless EVM block execution tool. It's stateless because in Ethereum nodes like geth, it doesn't need to maintain database of blocks to do a block execution or re-execution. The `evm-server` service transforms Block Specimens to Block Results entirely on the input of underlying capture of Block Specimen data.

#### Clone & Build evm-server

Clone [covalenthq/erigon](https://github.com/covalenthq/erigon/) that contains the fork for this particular stateless execution function, build the `evm-server` binary

#### Run the generated binary

```
$ ./build/bin/evm t8n --server.mode

  [INFO] [06-22|13:53:54.148] Listening port=3002
```

**Note**: `evm-server` occupies the port 3002 on your local at `http://evm-server:3002"`, so make sure this port in not occupied by any other service prior to start.

#### Run IPFS-Pinner

The IPFS-Pinner is an interface to the storage layer of the Covalent Network using a decentralized storage network underneath. Primarily it's a custom [IPFS (Inter Planetary File System)](https://ipfs.tech/) node with pinning service components for [Web3.storage](https://web3.storage/) and [Pinata](https://www.pinata.cloud/); content archive manipulation etc. Additionally there's support for fetching using `dweb.link`. It is meant for uploading/fetching artifacts (like Block Specimens and uploading Block Results or any other processed/transformed data asset) of the Covalent Decentralized Network.

#### Clone & Build ipfs-pinner

Clone [covalenthq/ipfs-pinner](https://github.com/covalenthq/ipfs-pinner/) and build the pinner server binary

```
git clone https://github.com/covalenthq/ipfs-pinner.git --depth 1
cd ipfs-pinner
make clean server-dbg
```

Set the environment variable required by&#x20;

&#x20;by getting&#x20;

&#x20;&

from web3.storage and adding it to an&#x20;

&#x20;file.

```
$ cat .envrc

export W3_AGENT_KEY="<<W3_AGENT_KEY>>"
export DELEGATION_PROOF_FILE_PATH="<<DELEGATION_PROOF_FILE_PATH>>"
```

#### Start the ipfs-pinner server

You should get the "w3 agent key" and "delegation proof file" from Covalent. After that, start the `ipfs-pinner` server.

```
./build/bin/server -w3-agent-key $W3_AGENT_KEY -w3-delegation-file $W3_DELEGATION_FILE

generating 2048-bit RSA keypair...done
peer identity: QmZQSGUEVKQuCmChKqTBGdavEKGheCQgKgo2rQpPiQp7F8

Computing default go-libp2p Resource Manager limits based on:
    - 'Swarm.ResourceMgr.MaxMemory': "17 GB"
    - 'Swarm.ResourceMgr.MaxFileDescriptors': 61440

Applying any user-supplied overrides on top.
Run 'ipfs swarm limit all' to see the resulting limits.

2023/04/20 12:47:49 Listening...
```

**Note**: `ipfs-pinner` occupies the port 3001 on your local at `http://ipfs-pinner:3001"`, so make sure this port in not occupied by any other service prior to start.

#### **Debugging: Repo Migrations**

In case of the following error

```
2023-03-10T08:46:18.621-0800	FATAL	ipfs-pinner	ipfs-pinner/pinner.go:29	error initializing ipfs node: ipfs repo needs migration, please run migration tool.
See https://github.com/ipfs/fs-repo-migrations/blob/master/run.md
Sorry for the inconvenience. In the future, these will run automatically.
```

Run the following migrations

```
wget https://dist.ipfs.tech/fs-repo-migrations/v2.0.2/fs-repo-migrations_v2.0.2_darwin-arm64.tar.gz
tar -ztvf fs-repo-migrations_v2.0.2_darwin-arm64.tar.gz
cd fs-repo-migrations/
./fs-repo-migrations
```

### &#x20;**Run Block Results Producer (BRP):**

BRP is the primary orchestrator and supervisor for all transformation pipeline processes that locates a source Covalent Network data object to apply a tracing/execution/transformational rule to and outputs a new object generated from using such a rule.

Both the source and output are available through a decentralized storage service such as a wrapped IPFS node. A transformation-proof transaction is emitted, confirming that it has done this work along with the output content ids (ipfs) access URL.

To define what these three components are:

**Source:** The Block Specimen that serves as an input to the Refiner. Proof transactions made earlier to a smart contract with the respective cids (content ids) are where the source is checked.

**Rule:** A transformation plugin (or server) that can act on the Block Specimen (source). These can be compared to blueprints that have been shown to produce the data objects needed. Furthermore, anyone can create these rules to get a desired data object. Rule (or server) versions thus need to exist, tied to the decoded block specimen versions they are applied on.

**Target:** The output generated from running the rule over the object that came from the source that is the block result.

**Steps to Run BRP:**

#### Clone Refiner

Clone the `refiner` repo

```
git clone https://github.com/covalenthq/refiner.git
cd refiner
git checkout main
```

Get your `BLOCK_RESULT_OPERATOR_PRIVATE_KEY` that has `DEV` tokens for Moonbase Alpha and is already whitelisted as Block Result Producer operator. Set the following environment variables for the local refiner by creating an `.envrc.local` file

* Copy paste the environment variables into this file

```
touch .envrc.local


export BLOCK_RESULT_OPERATOR_PRIVATE_KEY="BRP-OPERATOR-PK-WITHOUT-0x-PREFIX"
export NODE_ETHEREUM_MAINNET="<<ASK-ON-DISCORD>>"
export IPFS_PINNER_URL="http://127.0.0.1:3001"
export EVM_SERVER_URL="http://127.0.0.1:3002"
```

#### Load envrc file

Call to load `.envrc.local + .envrc` files with the command below and observe the following output, make sure the environment variables are loaded into the shell.

* Once the env vars are passed into the `.envrc.local` file and loaded in the shell with `direnv allow .`, build the `refiner` application for the `prod` env i.e moonbeam mainnet or `dev` env for moonbase alpha as discussed before.

```
direnv allow

direnv: loading ~/Calent/refiner/.envrc
direnv: loading ~/Covalent/refiner/.envrc.local
direnv: export +BLOCK_RESULT_OPERATOR_PRIVATE_KEY +ERIGON_NODE +IPFS_PINNER_URL +NODE_ETHEREUM_MAINNET
```

#### Get required dep for Refiner app

Get all the required dependencies and build the&#x20;

&#x20;app for the&#x20;

&#x20;environment (this points to Moonbase Alpha contracts). **Note**: Windows is currently not supported.

```
mix local.hex --force && mix local.rebar --force && mix deps.get
```

#### Build Refiner app

* **For moonbeam mainnet.**
* **For moonbase alpha.**

```
#moonbeam-mainnet
MIX_ENV=prod mix release

#moonbase-alpha
MIX_ENV=dev mix release

.
..
....
evm-server: http://127.0.0.1:3002
ipfs-node: http://127.0.0.1:3001
* assembling refiner-0.2.12 on MIX_ENV=dev
* skipping runtime configuration (config/runtime.exs not found)
* skipping elixir.bat for windows (bin/elixir.bat not found in the Elixir installation)
* skipping iex.bat for windows (bin/iex.bat not found in the Elixir installation)
Release created at _build/dev/rel/refiner
    # To start your system
    _build/dev/rel/refiner/bin/refiner start
Once the release is running:
    # To connect to it remotely
    _build/dev/rel/refiner/bin/refiner remote
    # To stop it gracefully (you may also send SIGINT/SIGTERM)
    _build/dev/rel/refiner/bin/refiner stop
To list all commands:
   _build/dev/rel/refiner/bin/refiner
```

#### Start the Refiner

Start the `refiner` application and execute the proof-chain block specimen listener call which should run the Refiner pipeline pulling Block Specimens from IPFS using the cids read from recent proof-chain finalized transactions, decoding them, and uploading and proofing Block Results while keeping a track of failed ones and continuing (soft real-time) in case of failure.

The erlang concurrent fault tolerance allows each pipeline to be an independent worker that can fail (for any given Block Specimen) without crashing the entire pipeline application. Multiple pipeline worker children threads continue their work in the synchronous queue of Block Specimen AVRO binary files running the stateless EVM binary (`evm-server`) re-execution tool.

```
#moonbeam-mainnet
MIX_ENV=prod mix run --no-halt --eval 'Refiner.ProofChain.BlockSpecimenEventListener.start()';

#moonbase-alpha
MIX_ENV=dev mix run --no-halt --eval 'Refiner.ProofChain.BlockSpecimenEventListener.start()';


..
...
refiner       | [info] found 1 bsps to process
ipfs-pinner  | 2023/06/29 20:28:30 unixfsApi.Get: getting the cid: bafybeiaxl44nbafdmydaojz7krve6lcggvtysk6r3jaotrdhib3wpdb3di
ipfs-pinner  | 2023/06/29 20:28:30 trying out https://w3s.link/ipfs/bafybeiaxl44nbafdmydaojz7krve6lcggvtysk6r3jaotrdhib3wpdb3di
ipfs-pinner  | 2023/06/29 20:28:31 got the content!
refiner       | [info] Counter for ipfs_metrics - [fetch: 1]
refiner       | [info] LastValue for ipfs_metrics - [fetch_last_exec_time: 0.001604]
refiner       | [info] Sum for ipfs_metrics - [fetch_total_exec_time: 0.001604]
refiner       | [info] Summary for ipfs_metrics  - {0.001604, 0.001604}
refiner       | [debug] reading schema `block-ethereum` from the file /app/priv/schemas/block-ethereum.avsc
refiner       | [info] Counter for bsp_metrics - [decode: 1]
refiner       | [info] LastValue for bsp_metrics - [decode_last_exec_time: 0.0]
refiner       | [info] Sum for bsp_metrics - [decode_total_exec_time: 0.0]
refiner       | [info] Summary for bsp_metrics  - {0.0, 0.0}
refiner       | [info] submitting 17586995 to evm http server...
evm-server   | [INFO] [06-29|20:28:31.859] input file at                            loc=/tmp/3082854681
evm-server   | [INFO] [06-29|20:28:31.862] output file at:                          loc=/tmp/1454174090
evm-server   | [INFO] [06-29|20:28:32.112] Wrote file                               file=/tmp/1454174090
refiner       | [info] writing block result into "/tmp/briefly-1688/briefly-576460747542186916-YRw0mRjfExGMk4M672"
refiner       | [info] Counter for bsp_metrics - [execute: 1]
refiner       | [info] LastValue for bsp_metrics - [execute_last_exec_time: 3.14e-4]
refiner       | [info] Sum for bsp_metrics - [execute_total_exec_time: 3.14e-4]
refiner       | [info] Summary for bsp_metrics  - {3.14e-4, 3.14e-4}
ipfs-pinner  | 2023/06/29 20:28:32 generated dag has root cid: bafybeic6ernzbb6x4qslwfgklveisyz4vkuqhaafqzwlvto6c2njonxi3e
ipfs-pinner  | 2023/06/29 20:28:32 car file location: /tmp/249116437.car
[119B blob data]
ipfs-pinner  | 2023/06/29 20:28:34 Received /health request: source= 127.0.0.1:34980 status= OK
ipfs-pinner  | 2023/06/29 20:28:34 uploaded file has root cid: bafybeic6ernzbb6x4qslwfgklveisyz4vkuqhaafqzwlvto6c2njonxi3e
refiner       | [info] Counter for ipfs_metrics - [pin: 1]
refiner       | [info] LastValue for ipfs_metrics - [pin_last_exec_time: 0.002728]
refiner       | [info] Sum for ipfs_metrics - [pin_total_exec_time: 0.002728]
refiner       | [info] Summary for ipfs_metrics  - {0.002728, 0.002728}
refiner       | [info] 17586995:48f1e992d1ac800baed282e12ef4f2200820061b5b8f01ca0a9ed9a7d6b5ddb3 has been successfully uploaded at ipfs://bafybeic6ernzbb6x4qslwfgklveisyz4vkuqhaafqzwlvto6c2njonxi3e
refiner       | [info] 17586995:48f1e992d1ac800baed282e12ef4f2200820061b5b8f01ca0a9ed9a7d6b5ddb3 proof submitting
refiner       | [info] Counter for brp_metrics - [proof: 1]
refiner       | [info] LastValue for brp_metrics - [proof_last_exec_time: 3.6399999999999996e-4]
refiner       | [info] Sum for brp_metrics - [proof_total_exec_time: 3.6399999999999996e-4]
refiner       | [info] Summary for brp_metrics  - {3.6399999999999996e-4, 3.6399999999999996e-4}
refiner       | [info] 17586995 txid is 0xd8a8ea410240bb0324433bc26fdc79d496ad0c8bfd18b60314a05e3a0de4fb06
refiner       | [info] Counter for brp_metrics - [upload_success: 1]
refiner       | [info] LastValue for brp_metrics - [upload_success_last_exec_time: 0.0031149999999999997]
refiner       | [info] Sum for brp_metrics - [upload_success_total_exec_time: 0.0031149999999999997]
refiner       | [info] Summary for brp_metrics  - {0.0031149999999999997, 0.0031149999999999997}
refiner       | [info] Counter for refiner_metrics - [pipeline_success: 1]
refiner       | [info] LastValue for refiner_metrics - [pipeline_success_last_exec_time: 0.0052]
```

#### Check logs

Check logs for any errors in the pipeline process and note the performance metrics in line with execution. Checkout the documentation on what is being measured and why [here](https://github.com/covalenthq/refiner/blob/main/docs/METRICS.md/).

```
tail -f logs/log.log
..
...
refiner       | [info] Counter for refiner_metrics - [pipeline_success: 1]
refiner       | [info] LastValue for refiner_metrics - [pipeline_success_last_exec_time: 0.0052]
refiner       | [info] Sum for refiner_metrics - [pipeline_success_total_exec_time: 0.0052]
refiner       | [info] Summary for refiner_metrics  - {0.0052, 0.0052}
```

Alternatively - check proof-chain logs for correct block result proof submissions and transactions made by your block result producer.

[For moonbeam.](https://moonscan.io/address/0x4f2E285227D43D9eB52799D0A28299540452446E)

[For moonbase](https://moonbase.moonscan.io/address/0x19492a5019B30471aA8fa2c6D9d39c99b5Cda20C).

**Note**:For any issues associated with building and re-compiling execute the following commands, that cleans, downloads and re-compiles the dependencies for `refiner`.

```
rm -rf _build deps && mix clean && mix deps.get && mix deps.compile
```

If you got everything working so far. Congratulations! You're now a Refiner operator on the CQT Network. Set up Grafana monitoring and alerting from links in the [additional resources](https://github.com/covalenthq/refiner#additional-resources/) section.

### Troubleshooting

To avoid permission errors with \~/.ipfs folder execute the following in your home directory.

```
sudo chmod -R 770 ~/.ipfs
```

To avoid netscan issue execute the following against ipfs binary application.

```
sudo chmod -R 700 ~/.ipfs
ipfs config profile apply server
```

To avoid issues with `ipfs-pinner` v0.1.9, a small repo migration for the local `~/.ipfs` directory may be needed from - [https://github.com/ipfs/fs-repo-migrations/blob/master/run.md](https://github.com/ipfs/fs-repo-migrations/blob/master/run.md).

For linux systems follow the below steps.

```
wget https://dist.ipfs.tech/fs-repo-migrations/v2.0.2/fs-repo-migrations_v2.0.2_linux-amd64.tar.gz
tar -xvf fs-repo-migrations_v2.0.2_linux-amd64.tar.gz
cd fs-repo-migrations
chmod +x ./fs-repo-migrations
./fs-repo-migrations
```

#### Bugs Reporting Contributions

Please follow the guide in docs [contribution guidelines](https://github.com/covalenthq/refiner/blob/main/docs/CONTRIBUTING.md/) for bug reporting and contributions.

### Scripts

In order to run the Refiner docker compose services as a service unit. The example service unit file in [docs](https://github.com/covalenthq/refiner/blob/main/docs/refiner-compose.service/) should suffice. After adding the env vars in their respective fields in the service unit file, enable the service and start it.

```
sudo systemctl enable refiner-compose.service
sudo systemctl start refiner-compose.service
```

**Note**:: To run docker compose as a non-root user for the above shown service unit, you need to create a docker group (if it doesn't exist) and add the user “blockchain” to the docker group.

```
sudo groupadd docker
sudo usermod -aG docker blockchain
sudo su - blockchain
docker run hello-world
```

#### Appendix

#### Run With Existing IPFS-Pinner Service

On a system where an `ipfs-pinner` instance is already running, use this modified `.envrc.local` and `docker-compose-mbase.yml`

```
export BLOCK_RESULT_OPERATOR_PRIVATE_KEY="BRP-OPERATOR-PK-WITHOUT-0x-PREFIX"
export NODE_ETHEREUM_MAINNET="<<ASK-ON-DISCORD>>"
export IPFS_PINNER_URL="http://host.docker.internal:3001"
export EVM_SERVER_URL="http://evm-server:3002"
```

**Note**: When passing the private key into the env vars as above please remove the `0x` prefix so the private key env var has exactly 64 characters.

```
version: '3'
# runs the entire refiner pipeline with all supporting services (including refiner) in docker
# set .env such that all services in docker are talking to each other only; ipfs-pinnern is assumed
# to be hosted on the host machine. It's accessed through http://host.docker.internal:3001/ url from
# inside refiner docker container.
services:
  evm-server:
    image: "us-docker.pkg.dev/covalent-project/network/evm-server:stable"
    container_name: evm-server
    restart: always
    labels:
      "autoheal": "true"
    expose:
      - "3002:3002"
    networks:
      - cqt-net
    ports:
      - "3002:3002"

  refiner:
    image: "us-docker.pkg.dev/covalent-project/network/refiner:stable"
    container_name: refiner
    links:
      - "evm-server:evm-server"
    restart: always
    depends_on:
      evm-server:
        condition: service_healthy
    entrypoint: >
      /bin/bash -l -c "
        echo "moonbeam-node:" $NODE_ETHEREUM_MAINNET;
        echo "evm-server:" $EVM_SERVER_URL;
        echo "ipfs-pinner:" $IPFS_PINNER;
        cd /app;
        MIX_ENV=prod mix release --overwrite;
        MIX_ENV=prod mix run --no-halt --eval 'Refiner.ProofChain.BlockSpecimenEventListener.start()';"
    environment:
      - NODE_ETHEREUM_MAINNET=${NODE_ETHEREUM_MAINNET}
      - BLOCK_RESULT_OPERATOR_PRIVATE_KEY=${BLOCK_RESULT_OPERATOR_PRIVATE_KEY}
      - EVM_SERVER_URL=${EVM_SERVER_URL}
      - IPFS_PINNER_URL=${IPFS_PINNER_URL}
    networks:
      - cqt-net
    extra_hosts:
      - "host.docker.internal:host-gateway"
    ports:
      - "9568:9568"

  autoheal:
    image: willfarrell/autoheal
    container_name: autoheal
    volumes:
      - '/var/run/docker.sock:/var/run/docker.sock'
    environment:
      - AUTOHEAL_INTERVAL=10
      - CURL_TIMEOUT=30

networks:
  cqt-net:
```

and start the `refiner` and `evm-server` services:

```
$ docker compose -f "docker-compose-mbeam.yml" up --remove-orphans

[+] Running 3/3
 ⠿ Network refiner_cqt-net  Created                                   0.0s
 ⠿ Container evm-server     Started                                   0.7s
 ⠿ Container refiner         Started                                   1.5s
```
