# learn-docker-network
the networking for containers to communicate with each other

## Docker Network
- Default network: bridge

## They are 6 network types (Driver) in Docker
1. `bridge` (default network driver in Docker)
1. `host` (use the host’s network namespace - no isolation between host and container)
1. `none` (no networking for a container)
1. `overlay` (connect multiple Docker daemons together)
1. `ipvlan` (connect containers to the network using L2)
1. `macvlan` (assign a MAC address to a container)


## Pre-requisite
- Docker installed
- if using Virtual Machine, make sure the network is set to `Bridged Adapter` connect to router home network

## `bridge` network

- `ip address show` check we have new network interface `docker0` with IP address
- `docker network ls` list all network

when we run a container, it will be attached to the `bridge` network by Default
- `docker run -d -it --rm --name panda busybox` run a container
- `docker run -d -it --rm --name redpanda busybox` run a container
- `docker run -d -it --rm --name blackpanda nginx` run a container

check all the containers are running `docker ps`
check is it create new network interface `ip address show`

check bridge link that connect to docker0
- `bridge link`

inspect the network bridge
- `docker network inspect bridge`

look at the name "panda", "redpanda", "blackpanda" in the network bridge. Also have the same IP address range
Also have DNS that copy from the host machine /etc/resolv.conf file put into the container

the docker0 network act like a switch that connect all the containers together

try to ping from one container to another containers
- `docker exec -it panda ping redpanda` or `docker exec -it panda sh` then
    - `ip address show` to check the IP address
    - `ping redpanda` to ping the container also ping via IP address  should be successful
    - `ping blackpanda` to ping the container also ping via IP address  should be successful
    - `ping anuchito.com` to ping the domain name should be successful because it copy from the host machine /etc/resolv.conf file
        - also `ip route` to check the default gateway as default route which is `docker0` ip address

how the `docker0` network get the container out to the internet?
- it the magic all *NAT masquerade* that happen in the `iptables` rules

We run `blackpanda` container with nginx web server we can access the web server from the host machine browser we need to export i we need to export it.
- `docker run -d -it --rm --name blackpanda -p 8080:80 nginx` run a container

## the user-defined network.
it is a bridge network but we can create our own network
docker suggest to create a user-defined network for the container to communicate with each other
- `docker network create zoo` create a bridge network name `zoo`
- `docker network ls` we will see the virtual bridge network link `zoo`
- `ip address show` we will see the new network interface `br-<network_id>` with new IP address

use the user-defined network `zoo` to run the container
- `docker run -d -it --rm --name lion --network zoo busybox` run a container
- `docker run -d -it --rm --name tiger --network zoo busybox` run a container

we get the network isolation between the `zoo` network and the `bridge` network they will not able to communicate with each other

we can ping from `lion` to `tiger` container
- `docker exec -it lion ping tiger` or `docker exec -it lion sh` then
    - `ping tiger` to ping the container also ping via IP address  should be successful


## `host` network
- `docker run -d -it --rm --name elephant --network host nginx` run a container we will not expose any port because it will use the host network
it run link regular application/process in the host machine even though it is in the container
if we want to run WireShark in the container we can use `host` networking


## `Macvlan` network
we just simply remove everythings and simply connect the container directly to physical network interface
it will look like the container just connect directly to the router/switch in home network
* they have their own *IP address* and *MAC address* in the home network

```sh
docker network create -d macvlan \
  --subnet=10.x.x.x/24 \  # the subnet of the home network
  --gateway=10.x.x.x \    # the gateway of the home network
  -o parent=eth0 \        # the physical network interface tigh MACVLAN to the pyhsical network interface
  zoovlan
```

```sh
docker run -d -it --rm --name zebra --network zoovlan \
    --ip 10.x.x.x \ # manually assign the IP address in the home network make sure it is not conflict with other devices
    --name zebra busybox

docker run -d -it --rm --name giraffe --network zoovlan \
    --ip 10.x.x.y \ # manually assign the IP address in the home network make sure it is not conflict with other devices
    --name giraffe busybox
```


and now *zebra* and *giraffe* connect directly to the home network
but the issue is the containers connect to same port in the home network because a physical LAN port can only have one MAC address
it have port security that only allow one MAC address to connect to the port
so we need to disable the port security in the router/switch to allow multiple MAC address to connect to the port
that is call `promiscuous mode` in the router/switch but it is not recommended to do that because it is a security risk

```sh
sudo ip link set eth0 promisc on
```
if it virtualbox we need to do it in the virtualbox setting

we can try to run new container in the same network
- `docker run -d -it --rm --name monkey --network zoovlan --ip 10.x.x.z busybox` run a container

note: the macvlan no DHCP server but docker will provide the docker default DHCP server to assign the IP address to the container
or we can use ip range to assign the IP address to the container

```sh
docker network create -d macvlan \
  --subnet=10.x.x.x/24 \
  --gateway=10.x.x.x \
  -o parent=eth0 \
  --ip-range=10.x.x.x/24 \
  zoovlan
```

this is macvlan bridge mode that connect to the home network

### Macvlan (802.1q) VLAN mode
it will create a VLAN sub-interface in the physical network interface
```sh
docker network create -d macvlan \
  --subnet=10.x.x.x/24 \
  --gateway=10.x.x.x \
  -o parent=eth0.10 \ # the VLAN sub-interface
  zoovlan.10
```

- `ip address show` we will see the new network interface `eth0.10` with new IP address


## `ipvlan` network
two mode L2 and L3
it allow the container share the same MAC address but different IP address

the default mode is L2 mode don't need to specify the mode
```sh
docker network create -d ipvlan \
  --subnet=10.x.x.x/24 \
  --gateway=10.x.x.x \
  -o parent=eth0 \
  zooipvlan
```

```sh
docker run -d -it --rm --name lion --network zooipvlan busybox
```
ping the lion container it will slow the same MAC address but different IP address
if we check `arb -a` we will see the same MAC address but different IP address

L3 mode it about IP address, routing. no more switching no more arp
we will create new network out of the thin air

we dont' need to specify gateway because the gateway will be parent interface gateway
```sh
docker network create -d ipvlan \
  --subnet=10.x.x.x/24 \
  -o parent=eth0 \
  -o ipvlan_mode=l3 \
  --subnet=10.x.y.x/24 \ # it will create more than one network
  zooipvlanl3
```

```sh
docker run -d -it --rm --name lion --network zooipvlanl3 --ip 10.x.x.x busybox
docker run -d -it --rm --name tiger --network zooipvlanl3 --ip 10.x.y.x busybox
```
we need to specify the IP address because it will create more than one network

ipvlan l3 can turn you host machine into a router because it can route the traffic between the containers
it need to enable the IP forwarding in the host machine
```sh
sudo sysctl -w net.ipv4.ip_forward=1
```

## `overlay` network
if we have multiple docker daemon running in different host machine and we want to connect the container between the host machine
we can talk to each other using overlay network
allow we to create rules to allow the container to communicate with each other


## `none` network
no networking for a container
the container will have only loopback interface

```sh
docker run -d -it --rm --name none --network none busybox
```
