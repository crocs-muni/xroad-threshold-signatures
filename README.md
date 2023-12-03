# Deploying threshold signatures into X-Road

[X-Road](https://x-road.global/) is a data exchange system used in many countries, for example Estonia and Finland. The [project code](https://github.com/nordic-institute/X-Road/)  is open-sourced. This tutorial acompanies the submitted publication: The Power of Many: Securing Organisational Identity Through Distributed Key Management. The goal of this tutorial is to explain how to setup and configure X-Road in order to employ threshold multiparty signatures.

<!-- ## Background --> 

<!-- The Members of the X-Road network exchange messages. Imagine a two-member scenario, where one --> 

## Tutorial

On a high-level the tutorial will help you set up:

1. Testing instance of the X-Road network with several members and their respective information systems.
1. Threshold signature platform that comprises.
1. Configure the platform to a particular X-Road member through PKCS#11 interface.


### 1. X-Road test network

This step is optional. In case you already have an X-Road instance with a Central Server (CS), Security Servers (SS) and Members' Information Systems connected you can omit this step.

X-Road can be deployed in various ways. One way to deploy it is to follow [this tutorial](https://www.youtube.com/watch?v=RV-Dq69yFVE&list=PL1hmZln4PWNWjy-7aCYrdr6yo3NDOvrT9) available on Nordic Institute for Interoperability Solutions YouTube channel. Although, the tutorial uses older version of X-Road, the setup of the virtual machines using LXD is still applicable. Once Central Server and Security Server is setup the following [configuration guide](https://nordic-institute.atlassian.net/wiki/spaces/XRDKB/pages/215875585/How+to+Configure+Central+Server+version+7.3.0#Table-of-contents) can be used. Once the Central Server and the Security Server are configured the instance can be tested as described [here](https://nordic-institute.atlassian.net/wiki/spaces/XRDKB/pages/4915774/How+to+Test+That+the+Security+Server+Is+Working+After+Completing+Installation+and+Configuration). For detailed instructions on installing CA and SS see: 
- [Central Server installation](https://github.com/nordic-institute/X-Road/blob/develop/doc/Manuals/ig-cs_x-road_6_central_server_installation_guide.md)
- [Security Server installation](https://github.com/nordic-institute/X-Road/blob/develop/doc/Manuals/ig-ss_x-road_v6_security_server_installation_guide.md)

The expected outcome at this point is four containers:
- `democa`
- `democs` with Central Server
- `demoss1` with Security Server
- `demoss2` with Security Server

### 2. Treshold signature platform

For the threshold signature platform we use [the MeeSign platform](https://meesign.crocs.fi.muni.cz).

### 3. Deploy
