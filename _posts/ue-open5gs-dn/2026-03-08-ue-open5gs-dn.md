---
layout: post
title: Setting UERANSIM to connect to DN through Open5GS
date: 2026-03-08 23:26:20 -0700
modified: 2026-03-12 13:14:35 -0700
description: A note on how to set up 5G CN using Open5GS and a remote RAN with UERANSIM.
tag: 
    - 5G
---
<!-- 
**Update History:**
- Mar 10, 2026: Fixed typos
- Mar 12, 2026: Added 5G SBA diagram  
  
--- -->

This post is a note on how to set up a 5G CN using Open5GS and UERANSIM for RAN on two separate hosts. The goal is pretty simple: establish a connection between UE and DN through UPF (i.e., UE &rarr; gNodeB &rarr; UPF &rarr; DN)
as illustrated in Figure 1 below.

<figure id="sba">
  <img src="/assets/img/ue-open5gs-dn/setup.png" alt="Open5GS and UERANSIM setup" />
  <figcaption>Figure 1. Open5GS and UERANSIM setup</figcaption>
</figure>

From here on, all the installations and configurations are done on **Ubuntu 24.04 LTS**.  
Let's start by running Open5GS first.

### Setting Up Open5GS 
Over the past few years, the installation procedure for [Open5GS](https://open5gs.org/open5gs/docs/guide/01-quickstart/) has not changed much and is quite straightforward.

```sh
$ sudo add-apt-repository ppa:open5gs/latest
$ sudo apt update
$ sudo apt install open5gs
```

By default, the core functions of Open5GS are configured to run on a single host. 
But we want to run a remote gNodeB and UE using UERANSIM, so we need to bind the Access and Mobility Function (**AMF**) to the IP address of the host runnning it. 
AMF is a core function in 5G SA that handles connection, mobility, and session management including UE registration and handover procedure.

- **/etc/open5gs/amf.yaml:**

```yaml
amf:
  ngap:
    server:
      - address: <HOST_IP>
...
```

_ngap_ represents the NGAP protocol used in N2 interface between gNodeB and AMF, and changing the loopback address to the host IP address allows the remote gNodeB to connect to the AMF.

There are other fields that you can configure, such as _MCC_ and _MNC_. The default value for each field is 999 and 70 respectively.
999 for MCC is used for internal private networks, and unless you have a specific reason to change it to another value, you can leave it as it is.

We should also configure the User Plane Function (**UPF**) to allow user packets to be transported from gNodeB to DN through the UPF.

- **/etc/open5gs/upf.yaml:**

```yaml
upf:
  gtpu:
    server:
        - addr: <HOST_IP>
...
```

Make sure to restart each service after making changes to the configuration files.

```sh
$ sudo systemctl restart open5gs-amfd.service
$ sudo systemctl restart open5gs-upfd.service
```

### Setting Up UERANSIM

On a separate host, we install [UERANSIM](https://github.com/aligungr/UERANSIM/wiki) to run a remote gNodeB and UE.

```sh
$ git clone https://github.com/aligungr/UERANSIM
$ cd UERANSIM
$ make
```

Configure gNodebB first:

- **UERANSIM/config/open5gs-gnb.yaml:**

```yaml
# List of AMF address information
amfConfigs:
  - address: <IP_Addr_of_Open5GS_host>
    port: 38412
```

Configure UE:

- **UERANSIM/config/open5gs-ue.yaml:**

```yaml
# List of gNB IP addresses for Radio Link Simulation
gnbSearchList:
  - <IP_Addr_of_UERANSIM_host>
```

Now run the gNodeB first:

```sh
$ build/nr-gnb -c config/open5gs-gnb.yaml
```

If successfully connected to AMF, you should see the following output:

```
[ngap] [info] NG Setup procedure is successful

```

You can check AMF log too:

```sh
$ tail -f /var/log/open5gs/amf.log
```

Now we run the UE:

```sh
$ build/nr-ue -c config/open5gs-ue.yaml
```

The following output indicates that the UE is successfully connected to gNodeB:

```
[nas] [debug] Sending Registration Complete
[nas] [info] Initial Registration is successful
[nas] [debug] Sending PDU Session Establishment Request
[nas] [debug] Configuration Update Command received
[nas] [debug] PDU Session Establishment Accept received
[nas] [info] PDU Session establishment is successful PSI[1]
```

Now we should be able to see a new NIC is present (_uesimtun0_).  
It means the user packets are encapsulated through GTP-U and transmitted to UPF.  
We can make sure by running commands like ping through the _uesimtun0_ interfact:

```sh
$ ping -I uesimtun0 8.8.8.8
```

Congrats! We have set up our own private 5G network!
Next, I plan to post another note about bringing COTS device on air to test real wireless communication, not just the simulation.