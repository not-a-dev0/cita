# Getting Started

## Dependencies

### System platform requirements

CITA is developed based on the stable version of Ubuntu 16.04 and runs robustly on this version.

It is recommended to use docker to compile and deploy CITA to ensure a consistent environment.

### Install docker

see [online information](https://docs.docker.com/install/).

### Get Docker Image

CITA docker image is hosted on [DockerHub](https://hub.docker.com/r/cita/cita-build/)

This can be obtained directly from the DockerHub using the `docker pull` command. See [online information](https://docs.docker.com/engine/reference/commandline/pull/).

For intranet environments, you can also use `docker save` and `docker load` command to deliver image.

### Get source code

Download CITA source code from Github repository, and switch to CITA source directory.

```shell
git clone https://github.com/cryptape/cita.git
cd cita
git submodule init
git submodule update
```

### Docker env and daemon

In the root directory of the source code, we provide `env.sh` script，which encapsulates docker-related operations.

Running this script with actual commands that you want to run inside docker container environment as arguments.

For example：

```shell
./env.sh make debug
```

This means running`make debug`in docker container.

Running`./env.sh` without any arguments will directly get a shell in docker container.

If container is already created by root user, running `./env.sh` without any arguments by a non-root user will get the following error:

```shell
$ ./env.sh
  error: failed switching to "user": unable to find user user: no matching entries in passwd file
```
We should keep same user all the time.

We also provided`daemon.sh`, same usage as`env.sh`，but run in background.

If there are some docker-related errors, you can try again after executing the following command：

```shell
docker kill $(docker ps -a -q)
```

## Compile

You can choose the compilation method according to your needs (Debug or Release)

```shell
./env.sh make debug
```

or

```shell
./env.sh make release
```

The generated file is under the`target/install`. You only need to operate under this directory in production environment.。

## Generate node configuration

Switch to release directory at first:

```shell
cd target/install
```

The`create_cita_config.py`in the release directory is used to generate the node configuration file, including the Genesis block configuration, node-related configuration, network connection configuration, and private key configuration.

The tool defaults to generate a Demo with 4 local nodes:

```shell
./env.sh ./scripts/create_cita_config.py create --super_admin "0x4b5ae4567ad5d9fb92bc9afd6a657e6fa13a2523" --nodes "127.0.0.1:4000,127.0.0.1:4001,127.0.0.1:4002,127.0.0.1:4003"
```

In the production environment, user needs to change the default configuration according to the actual situation.

Use`create_cita_config.py -h`to get detailed help information, allowing custom configurations to include:

* System administrator account
* Network list, in the format of`IP1:PORT1,IP2:PORT2,IP3:PORT3 ... IPn:PORTn`
* Blocking interval
* Check for repeated transactions after accumulating a certain amount of historical transactions
* System contract detailed parameter values
* Consensus node address

After the node initialization, the node configuration file will be generated in the release directory. The generated node directory is:

* test-chain/0
* test-chain/1
* test-chain/2
* test-chain/3

### Node commands

View node commands via `./bin/cita`.
```shell
Usage: cita <command> <node> [options]
where <command> is one of the following:
    { help | setup | start | stop | restart | ping
      top | backup | clean | logs | logrotate }

Run `cita help` for more detailed information.
```

## Run nodes

The commands of operation the nodes are the same. Take`test-chain/0`as an example.

1. Configure the node:

    ```shell
    ./env.sh ./bin/cita setup test-chain/0
    ```

2. Start the node：

    This command does not return normally, so it needs to run in the background.

    ```shell
    ./daemon.sh ./bin/cita start test-chain/0
    ```

3. Stop the node：

    ```shell
    ./env.sh ./bin/cita stop test-chain/0
    ```

4. Other operations

    use help for detailed information：

    ```shell
    ./env.sh ./bin/cita help
    ```

## Docker-compose quick start

1. Get the binaries and docker-compose configuration

    ```
    latest_release_tag=$(curl --silent "https://api.github.com/repos/cryptape/cita/releases/latest" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
    echo "latest release tag: $latest_release_tag"
    wget https://github.com/cryptape/cita/releases/download/$latest_release_tag/cita_secp256k1_sha3.tar.gz
    tar zxvf cita_secp256k1_sha3.tar.gz
    cp -r cita_secp256k1_sha3 cita_secp256k1_sha3_node0
    cp -r cita_secp256k1_sha3 cita_secp256k1_sha3_node1
    cp -r cita_secp256k1_sha3 cita_secp256k1_sha3_node2
    cp -r cita_secp256k1_sha3 cita_secp256k1_sha3_node3

    wget https://raw.githubusercontent.com/cryptape/cita/develop/tests/integrate_test/docker-compose.yaml
    ```

2. Start and stop nodes

    ```
    USER_ID=`id -u $USER` docker-compose up -d
    docker-compose down
    ```

3. Run commands inside the containers

    ```
    docker-compose exec node0 /usr/bin/gosu user /bin/bash
    ```

4. Fetch the logs

    The default logs of a container are generated by the `cita-chain` microservice.

    ```
    docker-compose logs -f
    ```

    You can also access logs of all microservices in the path where the directory is mounted in the container.

    ```
    tail -100f cita_secp256k1_sha3_node0/test-chain/0/logs/cita-jsonrpc.log
    ```

## Verification

***Need to be executed after the test environment is set up***


- Query the number of nodes.

    Request:

    ```shell
    ./env.sh curl -X POST --data '{"jsonrpc":"2.0","method":"peerCount","params":[],"id":74}' 127.0.0.1:1337
    ```

    Result:

    ```shell
    {
        "jsonrpc": "2.0",
        "id": 74,
        "result": "0x3"
    }
    ```

- Query the current block height.

    Request:

    ```shell
    ./env.sh curl -X POST --data '{"jsonrpc":"2.0","method":"blockNumber","params":[],"id":83}' 127.0.0.1:1337
    ```

    Result:

    ```shell
    {
        "jsonrpc": "2.0",
        "id": 83,
        "result": "0x8"
    }
    ```

    Return the block height, indicating that the node has started to block out normally.

## multiple chains

To plan related port configuration for deploy multiple chains on different servers, Reference [config_tool](../chain/config_tool).

Deploying multiple chains on the one server, in addition to planning port configuration, due to the `rabbmitmq` system service limitation, multiple chains can only be run in one docker. Generate a new chain based on the directory where the `test-chain` :

  ```shell
  ./env.sh ./scripts/create_cita_config.py create --super_admin "0x4b5ae4567ad5d9fb92bc9afd6a657e6fa13a2523" --chain_name test2-chain --jsonrpc_port 2337 --ws_port 5337 --grpc_port 6000 --nodes "127.0.0.1:8000,127.0.0.1:8001,127.0.0.1:8002,127.0.0.1:8003"
  ```
  
  Run test2-chain is the same as the test-chain, and can only be run in the same docker.

More APIs (such as contract calls, transaction queries),please check[RPC calls](../rpc_guide/rpc)。
