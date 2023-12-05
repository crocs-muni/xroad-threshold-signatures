# Deploying threshold signatures into X-Road

[X-Road](https://x-road.global/) is a data exchange system used in many countries, for example Estonia and Finland. The [project code](https://github.com/nordic-institute/X-Road/)  is open-sourced. This tutorial acompanies the submitted publication: The Power of Many: Securing Organisational Identity Through Distributed Key Management and walks you through a test setup similar to the one used in the paper. The goal of this tutorial is to explain how to setup and configure X-Road in order to employ threshold multiparty signatures.


## Tutorial

On a high-level the tutorial will help you set up:

1. Test instance of the X-Road network with several members and their respective information systems.
1. Threshold signature platform that comprises.
1. Configure the platform to a particular X-Road member through PKCS#11 interface.

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
- `demoss2` at `10.174.114.40` with Security Server (SS<sub>P</sub>) for a Provider Member.

Client and Provider are regular X-Road Members with the respective subsystems. Each container and the `host` need to be able to reach other on a network (the IP addresses can, of course, differ). From the containers the `host` is available at `10.174.114.1`.

#### Provider Information System

For the Provider Information System we can use a basic REST application. The [full example](https://flask-restful.readthedocs.io/en/latest/quickstart.html#full-example) Flask application from [Flask RESTful](https://flask-restful.readthedocs.io/en/latest/) package is a straightforward example. This application is expected to be run inside the `host` on such a port (e.g., `8080`) so that `demoss2` is able to connect to it (i.e., the firewall is setup accordingly). When Flask app is working, the following request from the `host` can be used for testing access to the Provider's IS directly:
```shell
$ curl http://localhost:8080/todos
{"todo1": {"task": "build an API"}, "todo2": {"task": "?????"}, "todo3": {"task": "profit!"}}
```

#### Client Information System


The Client's Information System can simply be simulated by an HTTP request through e.g. cURL - similarly, to how we tested the Provider's Information System with cURL. The following is a template that should work at this time if the values are substituted correctly:
```shell
$ curl -H 'X-Road-Client: {X-Road instance identifier}/{Client Member class}/{Client Member code}/{Client Member Subsystem code}' \
    -X GET -i "https://{Clients' Security Server IP or domain}/r1/{X-Road instance identifier}/{Provider Member class}/{Provider Member code}/{Provider Member Subsystem code}/{Provider Member Subsystem Service name}/{REST API endpoint}" -k
```

You can also test that the TODOS service is reachable from `demoss1` with:
```shell
root@demoss1~# curl 10.174.114.1:8080/todos
{"todo1": {"task": "build an API"}, "todo2": {"task": "?????"}, "todo3": {"task": "profit!"}}
```


### 2. Treshold signature platform

For the threshold signatures we use [the MeeSign platform](https://meesign.crocs.fi.muni.cz). In order to use it one needs to have the MeeSign server and the clients running. Follow the instructions to build the [MeeSign server](https://github.com/crocs-muni/meesign-server). To setup the clients follow [this README.md](https://github.com/quapka/meesign-client/tree/add-ptsrsap1) and then start the desirable number of signers (five in order to follow this example closely), use the [_always sign_](https://github.com/quapka/meesign-client/tree/add-ptsrsap1/meesign_core#always-sign-example-client) application (i.e., the [`always_sign.dart`](https://github.com/quapka/meesign-client/blob/add-ptsrsap1/meesign_core/example/always_sign.dart)).  At this point you should have five Dart signing applications running as individual processes.

Next, the MeeSign signing group needs to be registered to the server through the Meesign server CLI. The same application that starts the server can be used to also manage it. Change directory to the `meesign-server` project and do the following:
```shell
$ cargo run --release -- get-devices
[389d61f973aad5c11d4ea25d1d9c0aac19a9e3e354bd7d2dc52cec3205c00e80] A (seen before 1s)
[7cf029210bb63070ea2a2a9f790d9defc9ff597995dadb8485a1b2347b2fc0d8] B (seen before 1s)
[de725ce1f62d90dd080d0e8c6cd31626f00e65975cfff14808a3dd79e2799a2b] C (seen before 1s)
[cd5ee7e9817f5c4d834e86e625c020b45ab3b75192cdc1fba45f54b68db38ec9] D (seen before 1s)
[52b81dbc19f7bd33255ab8bf47e252883974d1a04d8fbfeaf247cd8848b97733] E (seen before 1s)
```

Each line of the output corresponds to a single connected device (i.e., five devices in our case). The random string in the square brackets is the ID of the device. The name of the individual devices are `A, B, C, D` and `E`. To create a 3-out-of-5 signing group run the following:
```shell
cargo run --release -- request-group 3outof5alwayssign 3 sign_challenge 389d61f973aad5c11d4ea25d1d9c0aac19a9e3e354bd7d2dc52cec3205c00e80 7cf029210bb63070ea2a2a9f790d9defc9ff597995dadb8485a1b2347b2fc0d8 de725ce1f62d90dd080d0e8c6cd31626f00e65975cfff14808a3dd79e2799a2b cd5ee7e9817f5c4d834e86e625c020b45ab3b75192cdc1fba45f54b68db38ec9 52b81dbc19f7bd33255ab8bf47e252883974d1a04d8fbfeaf247cd8848b97733
```

The previous command will create a group called `3outof5alwayssign` that is able to sign any challenge. Implicitly, the group size is five (due to the number of devices' IDs used). And the threshold for signing is `3`. Unfortunately, the server does not support persistance for now. Therefore, closing the server at this point would require fully redoing the previous setup of the group (up to the restart of the individual signing applications).

```shell
$ cargo run --release -- get-devices
[3082010a02820101008c006cbc05c14437df26328447d3ac992c1923d936f4be30bfa5520395ce78555404cac30ddb63bf7aeb0006e345827a72e61d6a0075f413ae77ca07b75400e8d20c7140cf2efbe53fa981a39d46f3c230f68c71a9b05024f06d9b7b1fbb1730870bdc4dfc1ebf4856161e3335d5070adc09d483a32c61918b4068a14f11fedaa3640ff33d7e91cca307c690508fd35a8929db723a84f4d112675c5935dc1ae546d19b56bc4ec9e6c81146a5a766af539dc4745bf1a554d93bb60c43434a63189875fb5647f0ffcd758ee8e227febc7c9a3038267a96aa1ba48a89934f058de1f1b9c730e3a48c33c91cfc1f19f8fbadbbf7124ddf338ff6fcadd5d9458c8e110203010001] GroupName (2-of-2, 3)
```

Each line corresponds to a signing group. Due to the proof of concept solution for now, we need the server to have only a single group. The value in the square brackets corresponds to the public RSA key of the group. You can check the value of it by the following commands:
```shell
$ echo '3082010a02820101008c006cbc05c14437df26328447d3ac992c1923d936f4be30bfa5520395ce78555404cac30ddb63bf7aeb0006e345827a72e61d6a0075f413ae77ca07b75400e8d20c7140cf2efbe53fa981a39d46f3c230f68c71a9b05024f06d9b7b1fbb1730870bdc4dfc1ebf4856161e3335d5070adc09d483a32c61918b4068a14f11fedaa3640ff33d7e91cca307c690508fd35a8929db723a84f4d112675c5935dc1ae546d19b56bc4ec9e6c81146a5a766af539dc4745bf1a554d93bb60c43434a63189875fb5647f0ffcd758ee8e227febc7c9a3038267a96aa1ba48a89934f058de1f1b9c730e3a48c33c91cfc1f19f8fbadbbf7124ddf338ff6fcadd5d9458c8e110203010001' | xxd -r -p | openssl rsa -pubin -noout -text
Public-Key: (2048 bit)
Modulus:
    00:8c:00:6c:bc:05:c1:44:37:df:26:32:84:47:d3:
    ac:99:2c:19:23:d9:36:f4:be:30:bf:a5:52:03:95:
    ce:78:55:54:04:ca:c3:0d:db:63:bf:7a:eb:00:06:
    e3:45:82:7a:72:e6:1d:6a:00:75:f4:13:ae:77:ca:
    07:b7:54:00:e8:d2:0c:71:40:cf:2e:fb:e5:3f:a9:
    81:a3:9d:46:f3:c2:30:f6:8c:71:a9:b0:50:24:f0:
    6d:9b:7b:1f:bb:17:30:87:0b:dc:4d:fc:1e:bf:48:
    56:16:1e:33:35:d5:07:0a:dc:09:d4:83:a3:2c:61:
    91:8b:40:68:a1:4f:11:fe:da:a3:64:0f:f3:3d:7e:
    91:cc:a3:07:c6:90:50:8f:d3:5a:89:29:db:72:3a:
    84:f4:d1:12:67:5c:59:35:dc:1a:e5:46:d1:9b:56:
    bc:4e:c9:e6:c8:11:46:a5:a7:66:af:53:9d:c4:74:
    5b:f1:a5:54:d9:3b:b6:0c:43:43:4a:63:18:98:75:
    fb:56:47:f0:ff:cd:75:8e:e8:e2:27:fe:bc:7c:9a:
    30:38:26:7a:96:aa:1b:a4:8a:89:93:4f:05:8d:e1:
    f1:b9:c7:30:e3:a4:8c:33:c9:1c:fc:1f:19:f8:fb:
    ad:bb:f7:12:4d:df:33:8f:f6:fc:ad:d5:d9:45:8c:
    8e:11
Exponent: 65537 (0x10001)
```

The public key of the group serves as an identifier of the signing group and it will be needed in the next step.


### 3. Configure SS<sub>C</sub> to use threshold signatures

In short, the goal of this step is to connect the signing group to SS<sub>C</sub> to serve as the signing device for Client Member.

First, build the [Cryptoki interface library](https://github.com/quapka/cryptoki-bridge/tree/add-rsa) for exposing MeeSign groups as PKCS#11. Then take the resulting shared library `target/release/libcryptoki_bridge.so` and copy it to the Client's Security Server. If you are using LXD you can do so with a similar command:
```shell
$ lxc file push target/release/libcryptoki_bridge.so demoss1/home/xroad/
```

The following commands are done inside the Client's Security Server container. If you are using LXD you can connect to it using the following command:
```shell
$ lxc exec demoss1 /bin/bash
root@demoss1:~#
```

Check that the previously added `libcryptoki_bridge.so` file is owned by the `xroad` user. Install the support for [Hardware Tokens](https://github.com/nordic-institute/X-Road/blob/develop/doc/Manuals/ig-ss_x-road_v6_security_server_installation_guide.md#210-installing-the-support-for-hardware-tokens). Next, open the configuration for the `xroad-signer` service in `/etc/xroad/devices.ini` and add an entry for MeeSign - the tested configuration is available in this repository in `example-devices.ini`. Add the contents of `example-devices.ini` to `/etc/xroad/devices.ini`. To let `libcryptoki_bridge.so` know about the MeeSign server and signing group set up the following enviromental variables so that the `xroad-signer` service has them available (the configuration is described in more detail in [here](https://github.com/KristianMika/mpc-bridge/wiki/Cryptoki-Bridge#configuration)):
```shell
COMMUNICATOR_HOSTNAME - sets the meesign hostname, e.g. 10.174.114.1 (the host in this tutorial)
COMMUNICATOR_CERTIFICATE_PATH - provides the library with the path to the CA certificate (required)
GROUP_ID - the public key encoded as hex from the output for `get-groups`.
```

When setting up the MeeSign server the, one of the steps was to generate keys and certificates for the server. They will be generated in `{meesign-server-path}/keys`. Copy the `meesign-ca-cert.pem` from the MeeSign server into the SS<sub>C</sub> container, e.g. with:
```shell
$ lxc file push {meesign-server-path}/keys/meesign-car-cert.pem demoss1/home/xroad/
```
Make sure the file is owned by the `xroad` user. E.g. using:
```shell
root@demoss1:~# chown xroad:xroad -R /home/xroad
```

One place where you can set these variables is `demoss2/etc/xroad/services/signer.conf`. So, following this example add to the file the following lines:
```shell
COMMUNICATOR_HOSTNAME=10.174.114.1
COMMUNICATOR_CERTIFICATE_PATH=/home/xroad/meesign-ca-cert.pem
GROUP_ID=3082010a02820101008c006cbc05c14437df26328447d3ac992c1923d936f4be30bfa5520395ce78555404cac30ddb63bf7aeb0006e345827a72e61d6a0075f413ae77ca07b75400e8d20c7140cf2efbe53fa981a39d46f3c230f68c71a9b05024f06d9b7b1fbb1730870bdc4dfc1ebf4856161e3335d5070adc09d483a32c61918b4068a14f11fedaa3640ff33d7e91cca307c690508fd35a8929db723a84f4d112675c5935dc1ae546d19b56bc4ec9e6c81146a5a766af539dc4745bf1a554d93bb60c43434a63189875fb5647f0ffcd758ee8e227febc7c9a3038267a96aa1ba48a89934f058de1f1b9c730e3a48c33c91cfc1f19f8fbadbbf7124ddf338ff6fcadd5d9458c8e110203010001
```

Before you can use the group for signing you need to restart the `xroad-signer` service with:
```shell
root@demoss1:~# systemctl restart xroad-signer.service
```

Finally, go to the web interface for Client's Security Server and add the new key in the `Keys and certificates` for the Client, generate CSR and use it as normally.

At this point it should be possible to use the threshold signing key to sign a Client's request:
```shell
$ curl -H 'X-Road-Client: {X-Road instance identifier}/{Client Member class}/{Client Member code}/{Client Member Subsystem code}' \
    -X GET -i "https://{Clients' Security Server IP or domain}/r1/{X-Road instance identifier}/{Provider Member class}/{Provider Member code}/{Provider Member Subsystem code}/{Provider Member Subsystem Service name}/{REST API endpoint}" -k
```
