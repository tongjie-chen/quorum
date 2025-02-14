# Trace the algorithms by log

Added log statements at critical part of the algorithms with prefix
\"Proj-TJ\" so it is distinguishable.

To filter this part of log only, run `geth` with `2>&1 | grep "Proj-TJ"`
added to the end of command.

# Client program setup

The client program is in repo [GitHub -
tongjie-chen/quorum-client](https://github.com/tongjie-chen/quorum-client).
It\'s written HTML. Needs to manually set up nodes and then run. The
node set up process is in [Use QBFT -
GoQuorum](https://consensys.net/docs/goquorum/en/latest/tutorials/private-network/create-qbft-network/).

## Set up nodes

In the working directory,

``` {.bash org-language="sh"}
cd ~/tmp/
mkdir qbft-test3
cd qbft-test3
mkdir -p ./QBFT-Network/{Node-0,Node-1}/data/keystore
tree
cd QBFT-Network
npx quorum-genesis-tool --consensus qbft --chainID 1337 --blockperiod 5 --requestTimeout 10 --epochLength 30000 --difficulty 1 --gasLimit '0xFFFFFF' --coinbase '0x0000000000000000000000000000000000000000' --validators 2 --members 0 --bootnodes 0 --outputPath 'artifacts'
time=`ls artifacts`
mv artifacts/$time/* artifacts
cd artifacts/goQuorum
sed -i 's/<HOST>/127.0.0.1/g' static-nodes.json
```

Now manually change the ports in the `static-nodes.json` and then run
the following script.

``` {.bash org-language="sh"}
cd ~/tmp/qbft-test3/QBFT-Network/artifacts/goQuorum
for i in `seq 0 1`; do cp static-nodes.json genesis.json ../../Node-$i/data/; done
cd ..
for i in `seq 0 1`; do cd ./validator$i; cp nodekey* address ../../Node-$i/data; cp account* ../../Node-$i/data/keystore; cd ..; done
cd ..
for i in `seq 0 1`; do cd Node-$i; geth --datadir data init data/genesis.json; cd ..; done
```

After that, can run the nodes like the following,

``` {.bash org-language="sh"}
cd Node-0
export ADDRESS=$(grep -o '"address": *"[^"]*"' ./data/keystore/accountKeystore | grep -o '"[^"]*"$' | sed 's/"//g')
export PRIVATE_CONFIG=ignore
geth --datadir data \
    --graphql --graphql.vhosts=* \
    --networkid 1337 --nodiscover --verbosity 5 \
    --syncmode full --nousb \
    --istanbul.blockperiod 5 --mine --miner.threads 1 --miner.gasprice 0 --emitcheckpoints \
    --http --http.addr 127.0.0.1 --http.port 22021 --http.corsdomain "*" --http.vhosts "*" \
    --ws --ws.addr 127.0.0.1 --ws.port 32021 --ws.origins "*" \
    --http.api admin,trace,db,eth,debug,miner,net,shh,txpool,personal,web3,quorum,istanbul,qbft \
    --ws.api admin,trace,db,eth,debug,miner,net,shh,txpool,personal,web3,quorum,istanbul,qbft \
    --unlock ${ADDRESS} --allow-insecure-unlock --password ./data/keystore/accountPassword \
    --port 30321
```

``` {.bash org-language="sh"}
cd Node-1
export ADDRESS=$(grep -o '"address": *"[^"]*"' ./data/keystore/accountKeystore | grep -o '"[^"]*"$' | sed 's/"//g')
export PRIVATE_CONFIG=ignore
geth --datadir data \
    --networkid 1337 --nodiscover --verbosity 5 \
    --syncmode full --nousb \
    --istanbul.blockperiod 5 --mine --miner.threads 1 --miner.gasprice 0 --emitcheckpoints \
    --http --http.addr 127.0.0.1 --http.port 22022 --http.corsdomain "*" --http.vhosts "*" \
    --ws --ws.addr 127.0.0.1 --ws.port 32022 --ws.origins "*" \
    --http.api admin,trace,db,eth,debug,miner,net,shh,txpool,personal,web3,quorum,istanbul,qbft \
    --ws.api admin,trace,db,eth,debug,miner,net,shh,txpool,personal,web3,quorum,istanbul,qbft \
    --unlock ${ADDRESS} --allow-insecure-unlock --password ./data/keystore/accountPassword \
    --port 30322
```

Take notes of the http port number and account in
`Node-0/data/keystore/accountAddress` or the one in Node-1.

``` {.bash org-language="sh"}
cat ~/tmp/qbft-test3/QBFT-Network/Node-0/data/keystore/accountAddress
```

## Deploy contract

Deploy the contract ([kaleido-js/simplestorage.sol at master ·
kaleido-io/kaleido-js ·
GitHub](https://github.com/kaleido-io/kaleido-js/blob/master/deploy-transact/contracts/simplestorage.sol))
first by following the guide in [Deploy a contract -
GoQuorum](https://consensys.net/docs/goquorum/en/latest/tutorials/contracts/deploying-contracts/)
with solidity compiler.

``` {.bash org-language="sh"}
wget https://raw.githubusercontent.com/kaleido-io/kaleido-js/master/deploy-transact/contracts/simplestorage.sol -O /tmp/simplestorage.sol
```

Change compiler version from `^` to \"\>=\", assuming no breaking
changes.

``` {.bash org-language="sh"}
sed -i 's/\^/>=/g' /tmp/simplestorage.sol
solc -o /tmp --abi --bin /tmp/simplestorage.sol
```

Or just directly change the \"from\" account address and the http port
in the end of next command to deploy and get the contract address.

``` {.bash org-language="sh"}
curl -X POST --data '{"jsonrpc":"2.0","method":"eth_sendTransaction","params":[{"from":"0x824ea808df43a92c2ab01c6bb6c2ca444b1b64f4", "to":null, "gas":"0x24A22","gasPrice":"0x0", "data":"0x608060405234801561001057600080fd5b5060405161014d38038061014d8339818101604052602081101561003357600080fd5b8101908080519060200190929190505050806000819055505060f38061005a6000396000f3fe6080604052348015600f57600080fd5b5060043610603c5760003560e01c80632a1afcd914604157806360fe47b114605d5780636d4ce63c146088575b600080fd5b604760a4565b6040518082815260200191505060405180910390f35b608660048036036020811015607157600080fd5b810190808035906020019092919050505060aa565b005b608e60b4565b6040518082815260200191505060405180910390f35b60005481565b8060008190555050565b6000805490509056fea2646970667358221220e6966e446bd0af8e6af40eb0d8f323dd02f771ba1f11ae05c65d1624ffb3c58264736f6c63430007060033"}], "id":1}' -H 'Content-Type: application/json' http://localhost:22021
```

``` {.bash org-language="sh"}
curl -X POST --data '{"jsonrpc":"2.0","method":"eth_getTransactionReceipt","params":["0x69e995d938b7b906a3145948802eb8e8f6891cdbb592f4b78de551da6f276cf6"], "id":1}' -H 'Content-Type: application/json' http://localhost:22021
```

Take note of contract address.

## Run the client

After getting those information. You can run the client program with
needed information.

With the `set` in contract, if two out of two nodes are working fine,
you will see the transaction stored in a block number returned.

If one of the two nodes is down, it will return \"Time out\" and the
indicator of current block number will not increase.

# Understanding of QBFT algorithm

Quorum Byzantine fault tolerance (QBFT) algorithm is an consensus
algorithm to ensure the safe operation of distributed system.

It\'s in a bigger category of algorithm-state machine replication
([State machine replication -
Wikipedia](https://en.wikipedia.org/wiki/State_machine_replication)),
where each machine depends on input and have finite steps to end
execution. To ensure the correctness, if `f` of the maximum tolerance
for the number of failed or malicious nodes, then `f+1` correct results
are needed. In case of malicious attacks of size `f`, the number of
total nodes `n` to ensure correct operation of the network is at least
`3f+1`.

Previous algorithms have been proposed. The advantage of QBFT is to have
a low communication latency of `3` because the `3` states transfer from
pre-prepare to prepare to commit. The communication complexity is
`O(n^2)` because `n` node broadcasts to the other nodes about size of
`n`. This is most relevant in situation where the execution and
communication delay is not easy to predict.

A quorum is the number of `floor((n+f)/2)+1` in the paper
<https://arxiv.org/pdf/2002.03613v2.pdf>, but depends on situation in
the code whether to be `2f+1` or `2/3f`.

The concensus starts with `startQBFT()` in `backend.go`, then to
`core.Start()`, `startNewRound()`.

A node is calculated to be a \"commander\" (leader or proposer) by
`CalcProposer()`. Then send broadcast pre-prepare to let other nodes
start to work in normal operation. A node knows if it is the
\"commander\" by pattern matching with `event.Data.(type)`.

Normal operation is like a state machine. Without failure of scale
larger than `f+1`, the state of node changes from `StateAcceptRequest`,
`StatePreprepared` to `StatePrepared` and finally `StateCommitted`.
Every state ends in broadcast (gossip) messages to other nodes. The
message include a number code and rlp encoded payload. Payload are very
similar structured with CommonPayload and auxlilary information of state
such as `digest` for prepare and `seal` for commit. The messages are
handled by a handler with pattern matching the number code.

Because the communication delay may be infinite if a node is down or
erroneous outputs from nodes, the algorithms uses round change to keep
liveness. When a time limit is reached for the normal operation as
counted by a node, it will broadcast to change round. When receiving a
quorum of valid messages and round change is justified, the node will
change round and start the normal operation.

Because of the number specified by quorum, the validity of algorithms
can be confirmed and also correctness. Full proof can be seen at
<https://arxiv.org/pdf/2002.03613v2.pdf>.

# Technical considerations in writing the client

Due to limited familiarity with Kafka, I figured out to use rpc and then
found the direct rpc calls requires a lot of encoding. So used `web3.js`
for the client to call with URL.

With `web3.js`, web development framework may be used for this client
writing. But it requires a server to set up to run. So finally ended up
with vanilla JavaScript on HTML with slight CSS styling.
