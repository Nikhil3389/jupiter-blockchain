# Hyperledger Fabric UAT Setup Guide

## Overview

This Hyperledger Fabric template contains the following:
- **2 Organizations**
- **4 Peers per organization** (each peer with CouchDB)

This document outlines the steps to start a Hyperledger Fabric network on two EC2 instances using Docker Swarm for a UAT setup.

## Step 1: Launch EC2 Instances

Launch two EC2 instances with the following configuration:

- **AMI ID:** `ami-0b2ec65899cc867ef`
- **Ubuntu Version:** `20.04.1-Ubuntu`
- **Docker Version:** `24.0.7`
- **Docker Compose Version:** `1.29.2`
- **Go Version:** `go1.21.6`

### Setup Instructions

1. Clone the repository that contains the necessary configuration for the Fabric network.
2. Run the `infra.sh` file to set up the required infrastructure on both EC2 instances.

## Step 2: Configure Host Names

You need to modify the `/etc/hosts` file on both EC2 instances to include the following host names for proper networking:

```bash
127.0.0.1 localhost
127.0.0.1 orderer.example.com
127.0.0.1 peer0.org1.example.com
127.0.0.1 peer1.org1.example.com
127.0.0.1 peer0.org2.example.com
127.0.0.1 peer1.org2.example.com
127.0.0.1 orderer5.example.com
127.0.0.1 orderer2.example.com
127.0.0.1 orderer3.example.com
127.0.0.1 orderer4.example.com
127.0.0.1 couchdb-peer2
127.0.0.1 couchdb-peer1
127.0.0.1 couchdb-peer0
127.0.0.1 couchdb-peer3
```


## Step 3: Set Environment Variables
Export the required environment variables on both EC2 instances:

```bash
export IMAGE_TAG=2.5   # Example image version
export SYS_CHANNEL=byfn-sys-channel # Example system channel name
```



## Step 4: Set Up Docker Swarm
### Initialize Docker Swarm
On the Swarm Leader EC2, run:

```bash
docker swarm init
docker swarm join-token manager
```

Copy the output from the docker `swarm join-token` command and run it on the Member EC2 to join the Swarm as a manager.

### Create the Swarm Overlay Network
On the Swarm Leader EC2, create the network:

```bash
docker network create --driver overlay --attachable first-network
```
Verify the network:
```bash
docker network ls
```
## Step 5: Start CouchDB Containers

On both EC2 instances, start the CouchDB containers in parallel.
### Commands for Swarm Leader EC2

```bash 
docker-compose -f couch-db-0 up -d
docker-compose -f couch-db-1 up -d
```

### Commands for Member EC2:
```bash
docker-compose -f couch-db-2 up -d
docker-compose -f couch-db-3 up -d
```
## Step 6: Start the Hyperledger Fabric Network
Next, start the Fabric network services on both EC2 instances.

### Commands for Swarm Leader EC2:
```bash 
docker-compose -f host1.yaml up -d
docker-compose -f host2.yaml up -d
```

Alternatively, you can use the helper scripts:

```bash
./host1up.sh
./host2up.sh
```

### Commands for Member EC2:
```bash 
docker-compose -f host3.yaml up -d
docker-compose -f host4.yaml up -d
```
Alternatively, use the helper scripts:

```bash 
./host3up.sh
./host4up.sh
```

## Step 7: Create the Channel and Join Peers

To create a new channel and have peers join it, run the following commands:
```bash 
docker exec cli peer channel create -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
sleep 5
docker exec cli peer channel join -b mychannel.block
docker exec -e CORE_PEER_ADDRESS=peer1.org1.example.com:8051 -e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls/ca.crt cli peer channel join -b mychannel.block
docker exec -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp -e CORE_PEER_ADDRESS=peer0.org2.example.com:9051 -e CORE_PEER_LOCALMSPID="Org2MSP" -e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt cli peer channel join -b mychannel.block
docker exec -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp -e CORE_PEER_ADDRESS=peer1.org2.example.com:10051 -e CORE_PEER_LOCALMSPID="Org2MSP" -e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt cli peer channel join -b mychannel.block

```
### Update Anchor Peers:

```bash 
docker exec cli peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
docker exec -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp -e CORE_PEER_ADDRESS=peer0.org2.example.com:9051 -e CORE_PEER_LOCALMSPID="Org2MSP" -e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt cli peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

Alternatively, you can run:

```bash 
./mychannelup.sh
```

## Step 8: Check Channel List
To verify the channels on peers, run the following commands:
```bash 
docker exec cli peer channel list
```
For EC2-1 (peer0.org1.example.com):

```bash 
docker exec peer0.org1.example.com peer channel getinfo -c mychannel
```
For EC2-2 (peer0.org2.example.com):
```bash 
docker exec peer0.org2.example.com peer channel getinfo -c mychannel
```

## Step 9: Bring the Network Down
To stop the Fabric network and clean up, run:
```bash 
./cleanup.sh
```

## Step 10: Access CouchDB on the Browser
To access CouchDB, connect via a custom VPN using Wireguard.
### Steps to Use VPN:

- Install Wireguard - Download from this https://apps.apple.com/us/app/wireguard/id1451685025?mt=12
- Import Configuration - Use the configuration file sent to you (do NOT share with anyone else)
- Activate Configuration in Wireguard App.

After activating the VPN, you can access CouchDB via:
```bash 
http://<internal_ip>:5984/_utils/#login
```
