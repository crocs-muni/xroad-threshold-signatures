# Deploying threshold signatures into X-Road

[X-Road](https://x-road.global/) is a data exchange system used in many countries, for example Estonia and Finland. The [project code](https://github.com/nordic-institute/X-Road/)  is open-sourced. This tutorial acompanies the submitted publication: The Power of Many: Securing Organisational Identity Through Distributed Key Management and walks you through the test setup used in the paper. The goal of this tutorial is to explain how to setup and configure X-Road in order to employ threshold multiparty signatures.

<!-- ## Background --> 

<!-- The Members of the X-Road network exchange messages. Imagine a two-member scenario, where one --> 

## Tutorial

On a high-level the tutorial will help you set up:

1. Testing instance of the X-Road network with several members and their respective information systems.
1. Threshold signature platform that comprises.
1. Configure the platform to a particular X-Road member through PKCS#11 interface.

z
### 1. X-Road test network


This step is optional. In case you already have an X-Road instance with a Central Server (CS), Security Servers (SS) and Members' Information Systems connected you can omit this step. The tested X-Road version was >= 7.3.0.

#### Certification Authority, Central Server and Security Servers

X-Road can be deployed in various ways. One way to deploy it is to follow [this tutorial](https://www.youtube.com/watch?v=RV-Dq69yFVE&list=PL1hmZln4PWNWjy-7aCYrdr6yo3NDOvrT9) available on Nordic Institute for Interoperability Solutions YouTube channel. Although, the tutorial uses older version of X-Road, the setup of the virtual machines using LXD is still applicable. Once Central Server and Security Server is setup the following [configuration guide](https://nordic-institute.atlassian.net/wiki/spaces/XRDKB/pages/215875585/How+to+Configure+Central+Server+version+7.3.0#Table-of-contents) can be used. Once the Central Server and the Security Server are configured the instance can be tested as described [here](https://nordic-institute.atlassian.net/wiki/spaces/XRDKB/pages/4915774/How+to+Test+That+the+Security+Server+Is+Working+After+Completing+Installation+and+Configuration). For detailed instructions on installing CA and SS see: 
- [Central Server installation](https://github.com/nordic-institute/X-Road/blob/develop/doc/Manuals/ig-cs_x-road_6_central_server_installation_guide.md)
- [Security Server installation](https://github.com/nordic-institute/X-Road/blob/develop/doc/Manuals/ig-ss_x-road_v6_security_server_installation_guide.md)

The expected outcome at this point is four "containers" (the configuration in the paper used LXD containers) that all run on a single `host` computer:
- `democa` at `10.174.114.10` with the test Certification Authority, OCSP and TSA.
- `democs` at `10.174.114.20` with the Central Server and the following Security Servers connected.
- `demoss1` at `10.174.114.30` with the Security Server (SS<sub>C</sub>) for a Client Member.
- `demoss2` at `10.174.114.40` with Security Server (SS<sub>P</sub>)for a Provider Member.

Client and Provider are regular X-Road Members with the respective subsystems. Each container and the `host` need to be able to reach other on a network (the IP addresses can of course differ). From the containers the `host` is available at `10.174.114.1`.

#### Provider Information System

For the Provider Information System test we can use a basic REST application. The [full example](https://flask-restful.readthedocs.io/en/latest/quickstart.html#full-example) Flask application from [Flask RESTful](https://flask-restful.readthedocs.io/en/latest/) package can be used. This application is expected to be run inside the `host` on such a port (e.g., `8080`) so that `demoss2` is able to connect to it (e.g., set up the firewall, etc.). Once successfully setup, the following request from the `host` can be used for testing access to the Provider directly:
```bash
$ curl http://localhost:8008/todos
{"todo1": {"task": "build an API"}, "todo2": {"task": "?????"}, "todo3": {"task": "profit!"}}
```

#### Client Information System


The Client's Information System can simply be simulated by an HTTP request through e.g. cURL - similarly, to how we tested the Provider's Information System with curl. The following is a template that should work at this time:
```bash
$ curl -H 'X-Road-Client: {X-Road instance identifier}/{Client Member class}/{Client Member code}/{Client Member Subsystem code}' \
    -X GET -i "https://{Clients' Security Server IP or domain}/r1/{X-Road instance identifier}/{Provider Member class}/{Provider Member code}/{Provider Member Subsystem code}/{Provider Member Subsystem Service name}/{REST API endpoint}" -k
```


### 2. Treshold signature platform

For the threshold signatures we use [the MeeSign platform](https://meesign.crocs.fi.muni.cz). In order to use it one needs have a MeeSign server and the clients running. Follow the instructions to install the [MeeSign server](https://github.com/crocs-muni/meesign-server). To setup the clients follow [this README.md](https://github.com/quapka/meesign-client/tree/add-ptsrsap1) and then start the desirable number of signers (five in order to follow this example closely), use the [_always sign_](https://github.com/quapka/meesign-client/tree/add-ptsrsap1/meesign_core#always-sign-example-client) application (i.e., the [`always_sign.dart`](https://github.com/quapka/meesign-client/blob/add-ptsrsap1/meesign_core/example/always_sign.dart)).  At this point you should have five Dart signing applications running as individual processes.

Next, the MeeSign signing group needs to be registered to the server through the Meesign server CLI. The same application that starts the server can be used to also manage it. Change directory to the `meesign-server` project and do the following:
```bash
$ cargo run --release -- get-devices
[389d61f973aad5c11d4ea25d1d9c0aac19a9e3e354bd7d2dc52cec3205c00e80] A (seen before 1s)
[7cf029210bb63070ea2a2a9f790d9defc9ff597995dadb8485a1b2347b2fc0d8] B (seen before 1s)
[de725ce1f62d90dd080d0e8c6cd31626f00e65975cfff14808a3dd79e2799a2b] C (seen before 1s)
[cd5ee7e9817f5c4d834e86e625c020b45ab3b75192cdc1fba45f54b68db38ec9] D (seen before 1s)
[52b81dbc19f7bd33255ab8bf47e252883974d1a04d8fbfeaf247cd8848b97733] E (seen before 1s)
```

The random string in the square brackets is the ID of the device. The name of the individual devices are `A, B, C, D` and `E`. To create a 3-out-of-5 signing group run the following:
```bash
cargo run --release -- request-group 3outof5alwayssign 3 sign_challenge 389d61f973aad5c11d4ea25d1d9c0aac19a9e3e354bd7d2dc52cec3205c00e80 7cf029210bb63070ea2a2a9f790d9defc9ff597995dadb8485a1b2347b2fc0d8 de725ce1f62d90dd080d0e8c6cd31626f00e65975cfff14808a3dd79e2799a2b cd5ee7e9817f5c4d834e86e625c020b45ab3b75192cdc1fba45f54b68db38ec9 52b81dbc19f7bd33255ab8bf47e252883974d1a04d8fbfeaf247cd8848b97733
```

The previous command will create a group called `3outof5alwayssign` that is able to sign any challenge. Implicitly, the group size is five (due to the number of devices' IDs used). And the threshold for signign is `3`. Unfortunately, the server does not support persistance for now. Therefore, closing the server at this point would require fully redoing the previous setup of the group (up to the restart of the individual signing applications).


- do I need to mention pretzel?
- Dart signing applications
- create a MeeSign group
- cryptoki, can I provide a build?

### 3. Deploy

deploy cryptoki to demoss2
alter demoss2/etc/xroad/devices.ini, 
alter ENV vars in demoss2/etc/xroad/signer.conf
