---
title: Docker Networking
date: 2024-01-04 18:03:59 -0600
categories: [Docker]
tags: [docker]     # TAG names should always be lowercase
---
This post dives into various topics about **Docker** network defaults and configurations. It enables one to get acquainted with navigating the flow of traffic and to provide the necessary tools to help one debug network issues within the environment. Please refer to the official [documentation](https://docs.docker.com/network/) for additional background, if needed.

## Docker Defaults
**Docker**, by default, permits a container to talk to the outside! The default network configuration for a container uses the default `bridge` network driver. The network drivers for  **Docker** enables its "networking subsystem to be pluggable" [[1]](https://docs.docker.com/network/drivers/). Each time a new container spins up with the default network configuration, it gets placed into the default `bridge` network (OS dependent, you may see this as `docker0`). The container gets bridged into this network and can communicate with other containers in its network. Furthermore, the default network gets NAT'd behind the host's internet protocol (IP) address. This permits containers in the default network to acquire connections beyond the local default network. **Docker** uses [iptables](https://docs.docker.com/network/packet-filtering-firewalls/) to accomplish this. One can follow the chains and rules put in place by **Docker** with the following:

```bash
# View the filter table
detrain@detrain:~$ sudo iptables -nvL --line-numbers
# View the nat table
detrain@detrain:~$ sudo iptables -t nat -nvL --line-numbers
```

Let's dive into the output of these commands a bit more to get a feel for how the default rules for **Docker** are setup. I assume you can follow iptable chains, and so I jump around a bit to highlight some of the rules imposed by **Docker**.

The virtual interface *docker0* serves as the exit/entry point for the default `bridge` network. **Docker** has populated rules in the FORWARD chain in the filter table. This enables one to forward traffic out of the `bridge` network.

```bash
Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
2        0     0 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
3        0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
4        0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
5        0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
6        0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0           

Chain DOCKER (1 references)
num   pkts bytes target     prot opt in     out     source               destination         

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 DOCKER-ISOLATION-STAGE-2  all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
2        0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 DROP       all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
2        0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-USER (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

One can see, in the filter table's FORWARD chain, rule **3** accepts the forwarding of RELATED, and ESTABLISHED connections going out on the interface *docker0*. This allows outbound traffic to return back into the `bridge` network, as any traffic going out will have related or an already established connection in regards to iptables. Next we see that the next, rule **4**, in the FORWARD chain relates to any traffic going into the virtual interface *docker0* will jump to the DOCKER chain to be further processed, in my case I do not have any published ports so this chain is empty. However, whenever you do publish a port you can see the rule get populated! For example:

```bash
detrain@detrain:~$ docker run -d --rm -p 80:80 nginx
```

> When one publishes a port, use the `-p` option with `docker run`. `-p` always takes the following form `host_port:container_port`. 
{: .prompt-info }

And now if we check the filter table's DOCKER chain, we can see the new rule that forwards traffic hitting the host's port 80 to the nginx container port 80. To explain this one step further, any traffic going out on docker0 that is not docker0 with a destination port 80, gets forwarded to 172.17.0.2.

```bash
Chain DOCKER (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 ACCEPT     tcp  --  !docker0 docker0  0.0.0.0/0            172.17.0.2           tcp dpt:80
```

We have investigated how traffic gets forwarded to the published socket, now lets investigate how the packet's fields get modified to enable the communication (hint: it has to do with NAT). I left the nginx container running from before so we can see how publish affects the iptables NAT table.

```bash
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        1    48 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           
2        0     0 MASQUERADE  tcp  --  *      *       172.17.0.2           172.17.0.2           tcp dpt:80

Chain DOCKER (2 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0           
2        0     0 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.2:80
```

First we can see that the PREROUTING chain will send all traffic to the DOCKER chain. Anything going into the *docker0* interface we just return, but the next rule relates to our published port! Anything coming on an interface that is not *docker0* for TCP port 80 gets its layer 3 (ip) / layer 4 (tcp) fields set to the nginx IP/port. 

Next lets look at the POSTROUTING chain. We can see the first rule is a MASQUERADE. Any traffic going out an interface that is not *docker0* and is in the network 172.17.0.0/16 (the default `bridge` network ID) will get NAT'd going out. If you're unfamiliar with iptables MASQUERADE I recommend a quick google, but this lets us NAT out of the network, which enables containers to reach out. And recall the FORWARD chain from the filter table, rule **3** lets all of the traffic MASQUERADE'd out return!

## Docker Network CLI Management
Below are some useful network management commands. Feel free to play aroud with them, but below is a quick demo to illustrate some of their use cases.

```bash
detrain@detrain:~$ docker network ls
```

```bash
detrain@detrain:~$ docker network inspect <network>
```

```bash
detrain@detrain:~$ docker network create --driver <driver>
```

```bash
detrain@detrain:~$ docker network connect
```

```bash
detrain@detrain:~$ docker network disconnect
```

> Don't forget our trusty friend `--help`!
{: .prompt-tip }

## Demonstration
This demo will showcase how to create a new custom network, and show one how to connect a container to a new network.

1. First, create a new network called new_custom_network. By not specifiying `--driver`, **Docker** defaults to the bridge driver.

    ```bash
    detrain@detrain:~$ docker network create new_custom_network
    ```

    One can see the new network via the command below.

    ```bash
    detrain@detrain:~$ docker network ls
    ```

2. Second, create a container on the new network.

    ```bash
    detrain@detrain:~$ docker run -d --rm --name hostA --network new_custom_network nginx
    ```

3. Third, create another container on the default bridge network. Normally, we could just add it the `new_custom_network` network, but for demonstration purposes let us say I forgot.

    ```bash
    detrain@detrain:~$ docker run -d --rm --name hostB -it busybox sh
    ```

    At this point we have two containers, hostB on network `bridge`, and hostA on network `new_custom_network`. These networks are isolated and so they cannot talk to each other. As a fun little challenge, do NOT take my word this is the case. Prove it!

    > Hint think iptables.
    {: .prompt-info }

4. To enable the communication between the two containers we need to connect one host to the other host's network. The default `bridge` network does not have any DNS capabilites, we can still communicate if we were to bring hostA to the `bridge` network, but in **Docker** it makes more sense to leverage its DNS capabilites as container lives may be short it is better to target the name of the container in the event that a container gets respawned. So let us bring hostB into the `new_custom_network` to show off a few commands!

    Connect hostB into the network of which hostA resides.

    ```bash
    detrain@detrain:~$ docker network connect new_custom_network hostB
    ```

    To confirm its success we can directly attempt to communicate with hostA, but instead let us inpsect the network to see the entry. I will `...` some of the output to keep this short and sweet. Below we can see the "Containers" key of the json output and verify hostA and hostB exist.

    ```bash
    detrain@detrain:~$ docker network inspect new_custom_network 
    ...
            "Containers": {
                "d6a4ac143fff5f7078db55ac709c2e8215a9296a7c9a5022af0916aee95e4006": {
                    "Name": "hostB",
                    "EndpointID": "e16b23db8ff3e52e47781fdbc078b9525f2e4edb28ce55dda771aa552284e589",
                    "MacAddress": "02:42:ac:12:00:03",
                    "IPv4Address": "172.18.0.3/16",
                    "IPv6Address": ""
                },
                "fbe6868459566de1dc64274ce8f687b6f20c863290ddb8789d3c9cff98e35bc4": {
                    "Name": "hostA",
                    "EndpointID": "b11cd8ca005fe0cb7ad47c3c2d4664bcb4181e8b77323f5da7b8cf6da371ad07",
                    "MacAddress": "02:42:ac:12:00:02",
                    "IPv4Address": "172.18.0.2/16",
                    "IPv6Address": ""
                }
            },
    ...
    ```

    We see the two containers reside in the same network! And to prove we have enabled communications between the two let us attempt to ping!

    ```bash
    docker container exec -it hostB ping hostA
    ```

## Summary
We dived into some of the **Docker** default iptables rules and how we can look through them to identify the various flows of network traffic. In addition, one should be comfortable with creating their own custom network and understand how to connect/disconnect a container from a network.

