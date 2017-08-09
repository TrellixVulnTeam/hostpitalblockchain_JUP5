﻿# Hyperledger v1.0 Network Setup
 **Note** -Kindly ensure that the VM has a proxyless access (all http/https ports are open)
 ________________________________________________________
 ### VM Specification
- **OS version :** Ubuntu 14.04.
- **RAM:** 4GB/8GB.
- **Docker Version:** 17.x.
- **Docker-Compose:** 1.8 or above.

 ________________________________________________________
 **Note :** If you dont want to make any changes in the network setup and want to try using SDK, then you can just follow step 2 and 3 to install the required hyperledger images and tools and execute the docker-compose-e2e-template file which will start 4 peers + 2 CA (2 peers and 1 certificate authority for each org), and 1 ordering service node. For SDk go to step 11
 
 ### Steps For Setting Up Hyperledger V1.0 Network
 
**1.** Run the command " cd / " <br />
**2.** Using curl command download all the required hyperlegder docker images and necessary tools <br/>
 ```sh 
    curl -sSL https://goo.gl/iX9dek | bash 
 ```
This will download binaries such as cryptogen,configtxgen,configtxlator, and peer in you /bin folder. Kindly check for binaries before you proceed<br/>

**3.** Clone the repository on your local machine using the command <br/>
  ```sh 
      git clone https://github.com/shubhamvrkr/network-setup 
  ```
**4.** Next step is to generate some crytographic material(certificates and signing keys) for organizations using cryptogen tool <br/>
Run the command to generate cryptographic material
```sh
   cd /network-setup 
   export FABRIC_CFG_PATH=$PWD
   cryptogen generate --config=./crypto-config.yaml
 ```
This command will take crypto-config.yaml as an input and will generate cryptographic material for one orderer organization and    two peer organizations. Kindly make the nessary changes to the file if you intend to generate cryptographic material for more than two orgs or if you want to change the organization structure <br/>

For more information on crypto-config.yaml go to [cryptogen](https://hyperledgerfabric.readthedocs.io/en/latest/build_network.html#crypto-generator)

 **5.** After all the necessary certificates are been generated, Next step is to generate a genesis block that contains all the necessary information about the org certificates(i.e root ca certs, admin certs and tlsca certs) used by the ordering service to validate incoming requests(e.g channel creation request)<br/>
  
  Run the command to generate genesis block
  ```sh
    export FABRIC_CFG_PATH=$PWD
    configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block
  ```
   This command will read configtx.yaml from current directory and will generate the genesis block from the profile specified (TwoOrgsOrdererGenesis). The genesis block will be created at the specied path(i.e ./channel-artifacts/genesis.block). The genesis file will be in protobuf format (type = Block), to convert into JSON format kindly see: [configtxlator tool](https://hyperledger-fabric.readthedocs.io/en/latest/configtxlator.html). By default profile includes MSP information of two orgs (Org1 and Org2) and Orderer mode as solo. If you want to include more orgs in the network or want to change orderer type to kafka or any setting related to the orderer than this file should be changed<br/>

For more information on configtx.yaml go to [configtxgen](https://hyperledger-fabric.readthedocs.io/en/latest/configtx.html)
   
   
**6.** Once the genesis block is generated, Its time to update the docker compose file.<br/>
- ###Configuing Certificate Authority
   - Open docker-compose-e2e-template.yaml.
   - In the services section, go to ca0 section
   - This section starts the certificate authority for organization(e.g Org1)
   - For FABRIC_CA_SERVER_TLS_CERTFILE give the path of the root ca certificate (e.g /crypto-config/peerOrganizations/org1.example.com/ca/ca-cert.org1.example.com.pem)
   - For FABRIC_CA_SERVER_TLS_KEYFILE give the path of the private key for root ca (e.g /crypto-config/peerOrganizations/org1.example.com/ca/*_sk). Private key will have _sk suffix to the file
   - In the command provide path for --ca.certfile which will be same as FABRIC_CA_SERVER_TLS_CERTFILE and --ca.keyfile which will be same as FABRIC_CA_SERVER_TLS_KEYFILE
   - The path for the above flags should be from the container perspective, so you might have to mount the volume of your root ca  directory to the container
   - **Note** If you setup has more than one CA, then make this changes for each CA services
   
- ###Configuing Orderer Service
   - Open base/docker-compose-base.yaml file.
   - In the services section, go to  orderer.example.com section
   - This section starts the ordering service
   - The name of the orderer container should be same as that of Domain name (recommended) specified for Domain in crypto-config.yml  while generating certificates for orderer node.
   - Mount the volume containing the genesis block to the orderer container
   - Mount the volume containing the certificates (tls & msp) for the orderering service node (e.g /crypto-  config/ordererOrganizations/example.com/orderers/orderer.example.com/msp)
   - For ORDERER_GENERAL_GENESISFILE flag, give the mounted genesis.block file from the container
   - For ORDERER_GENERAL_LOCALMSPID flag, give the MSP Id of the Ordering service which will be Name+"MSP", the name if the name we specified in the crypto-config file for orderer organization.
   - For ORDERER_GENERAL_TLS_PRIVATEKEY flag, give the path of the tls private key
   - For ORDERER_GENERAL_TLS_CERTIFICATE flag, give the path of the tls certificate key
   - For ORDERER_GENERAL_TLS_ROOTCAS flag, give the path of the tls root ca cert.
   - ALl the certificates will be present in the /crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls directory.

- ###Configuing Peer Service
   - Open base/docker-compose-base.yaml file.
   - In the services section, go to  peer0.org1.example.com section
   - Mount the volume containing the certificates (tls & msp) for the peer service node (e.g /fabric-test/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp)
   - change CORE_PEER_ADDRESS and CORE_PEER_LOCALMSPID which will be the MSP ID of the org.
   - Mount the peer's tls path from host to the container (e.g /fabric-test/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls), Kindly dont change the container path while mounting since the certificates path are refereed in the base/peer-base.yaml
   - Mount the peer's msp path from host to the container (e.g /fabric-test/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp) , Kindly dont change the container path while mounting since the certificates path are refereed in the base/peer-base.yaml
   -**Note**: Make alll the necessay changes for all peer services

**7.** After the neccessary changes have been made to the docker compose file, run the docker compose files using the command <br/>
   ```sh 
      docker-compose -f docker-compose-e2e-template up -d
  ```
 This command will execute the docker compose file and start listed peers,ca and ordering service.<br/>
 Run  the following command  to list all the docker containers and make sure all the containers are up and running. Proceed only if all the containers are up<br/>
 ```sh
    docker ps -a 
 ```
 
 **8.** Next Step is to create the channel artifacts (i.e channel.tx), This file contains the config object which is signed by the admin who sends the channel creation request to the ordering service <br/>
 
 Run the below command to generate the channel.tx file<br/>
 ```sh 
 configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME
 ```
 This command will read configtx.yaml from current directory and will generate the channel.tx from the profile specified (TwoOrgsChannel). The channel.tx will be created at the specied path(i.e ./channel-artifacts/channel.tx). The channel.tx will be in protobuf format (type = Envelope), to convert into JSON format you can use the configtxlator tool. Channel.tx files contains the orgs that are authorized to join the channel.By defaults two orgs are mentioned in the profile. If you want to add more orgs to the channel than change TwoOrgsChannel profile<br/>
 
  
 **9.** Once the channel.tx file is created ,it's time to send channel create request to the orderer. We will be using SDK to initiiate all request.<br/>
 
 **10.** Go to the Server folder and open server.js , All the required node modules are already present in the directory. If some additional packages are required install them.
 
 **11.** First step is to create the channel using the channel.tx which we have create, server.js file contains createChannel() method which creates the channel.
 The create channel method uses channel name that was specifed while creating channel.tx file, the orderer url and its tlsca certificate, Kindly go through the function and make necessary changes and once done run the server.js using command 
 ```sh
        node server.js
 ```
 Check the console for the status of the channel creation request
 
 **12.** Next step is to make join channel request to the peers of the organization. This command has to be run by admin of each org in the channel to join their respective peer.So if the channel includes two orgs , two times the joinchannel has to be called by admin of each org. Kindly go through the function and make necessary changes and once done run the server.js. Check the console for the status of the channel creation request
 
**13.** Once the peers of required orgs have joined the channel, you can run commands such as getallChannels , getChannelInfo to get the channel information. Dont forget to make the necessary changes to the functions in the server.js file

**14.** For more SDK fucntions go to https://fabric-sdk-node.github.io/
