## Hyperledger Fabric First Network(1.4.8) 

This is a rework of the BYFN script executed to create First-Network in Fabric Samples. We won't be running the script here instead we'll setup the network manually by running all commands step by step. Some changes are made to enhance the setup in a better way.
Changes made to official fabric-samples are listed below.

1) Chaincode folder and PWD in CLI are changed at lines 83 and 87 in docker-compose-cli file
2) Added a new script changeEnv.sh in order to change ENV variables faster during various peer execution commands

## STEP 1 Generation of crypto / certificate
$ cd first-network
$ cryptogen generate --config=./crypto-config.yaml 

## STEP 2 Generation of Configuration Transaction

a) Command to generate the genesis block

$ export FABRIC_CFG_PATH=$PWD
$ configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block

b) Command to generate the channel transaction

$ export CHANNEL_NAME=mychannel
$ configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME

c) Command to generate the transaction for the Anchor Peer in each Peer Organizations

$ configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP
$ configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP

## STEP 3 Bring up the nodes according to docker-compose file
$ docker-compose -f docker-compose-cli.yaml up -d

## STEP 4 Use CLI to setup the First Network
$ docker exec -it cli bash
Now all commands form here on will be executed in CLI container terminal

a) Create the channel or the mychannel.block artifact

$ export CHANNEL_NAME=mychannel
$ peer channel create -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
b) Join all the peers to the channel or mychannel.block file
Since all the environment variables are by default set to Peer0.Org1 in the CLI container during it instantiation we have to pass appropriate ENV variables to run commands for other peers.Now passing long env vars is very cumbersome so created a script file changeEnv.sh which will aid in changing Peer ENV vars for various execution of Peer Commands

For peer0.org1 (the default node),
$ peer channel join -b mychannel.block

For peer1.org1, we change environment variables in the script changeEnv.sh by uncommenting variables for Peer1.org1 and commenting out the rest
$ . ./changeEnv.sh
$ peer channel join -b mychannel.block
 
For peer0.org2, we change environment variables in the script changeEnv.sh by uncommenting variables for Peer0.org2 and commenting out the rest
$ . ./changeEnv.sh
$ peer channel join -b mychannel.block

For peer1.org2, we change environment variables in the script changeEnv.sh by uncommenting variables for Peer1.org2 and commenting out the rest
$ . ./changeEnv.sh
$ peer channel join -b mychannel.block

c) Update the Anchor Peer on each organization

For peer0.org1 (the default node),
$ peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile $ORDERER_CA

For peer0.org2 (Change the env vars accordingly)
$ . ./changeEnv.sh
$ peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile $ORDERER_CA

## STEP 5 Use CLI to install and instantiate the chaincode

a) Install Chaincode on peer0.org1 and peer0.org2

On peer0.org1 (default node),
$ peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/

On peer0.org2 (Change the env vars accordingly),
$ . ./changeEnv.sh
$ peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/

b) Instantiate Chaincode on peer0.org2(Change the env vars accordingly),
$ . ./changeEnv.sh
$ peer chaincode instantiate -o orderer.example.com:7050 --tls --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "AND ('Org1MSP.peer','Org2MSP.peer')"

b) Query value of 'a' on peer0.org1(Change the env vars accordingly),
$ . ./changeEnv.sh
$ peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'

## STEP 6 Use CLI to invoke the chaincode

a) Invoke the chaincode (equivalent to issue a transaction) from peer0.org1.We need to pass Peer Address and root Certificate of both peers
$ . ./changeEnv.sh
$ peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["invoke","a","b","10"]}'

b) Install chaincode on peer1.org2,
$ . ./changeEnv.sh
$ peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/

c) Query the value a from peer1.org2,
$ peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'

## STEP 7 Tear down all the setup
$ exit
$ docker-compose -f docker-compose-cli.yaml down
$ docker rm $(docker ps -aq)
$ docker network prune
$ docker volume prune