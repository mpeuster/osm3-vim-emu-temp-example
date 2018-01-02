# osm3-vim-emu-temp-example
Temporary example for OSM r3 to vim-emu integration.

This repo is only made temporary available. It gives an example on how to use the latest OSM THREE release with vim-emu by deploying an simple example network service with two VNFs (`pingpong`). This example will be moved to the official vim-emu repo. once it is finalized.

## Example service: `pingpong`

### Source descriptors

* Ping VNF (default `ubuntu:trusty` Docker container): `ping_vnf/`
* Pong VNF (default `ubuntu:trusty` Docker container): `pong_vnf/`
* Network service descriptor (NSD): `pingpong_ns/`

### Pre-packed VNF and NS packages

* Ping VNF: `ping.tar.gz`
* Pong VNF: `pong.tar.gz`
* NSD: `pingpong_nsd.tar.gz`

## Install OSM

Install OSM rel. THREE using the pre-build LXD images:

```sh
$ ./install_osm.sh --lxdimages
```

## Install / build / start vim-emu

```sh
# build the vim-emu Docker container
$ docker build -t vim-emu-img .

# run vim-emu in daemon mode
$ docker run --name vim-emu -t -d --rm --privileged --pid='host' -v /var/run/docker.sock:/var/run/docker.sock vim-emu-img python examples/osm_default_daemon_topology_2_pop.py

# watch vim-emu outputs ( do in another terminal window)
$ docker logs -f vim-emu

# check if the emulator is running in the container
$ docker exec vim-emu vim-emu datacenter list
+---------+-----------------+----------+----------------+--------------------+
| Label   | Internal Name   | Switch   |   # Containers |   # Metadata Items |
+=========+=================+==========+================+====================+
| dc2     | dc2             | dc2.s1   |              0 |                  0 |
+---------+-----------------+----------+----------------+--------------------+
| dc1     | dc1             | dc1.s1   |              0 |                  0 |
+---------+-----------------+----------+----------------+--------------------+
```

## Attach OSM to vim-emu

```sh
# get vim-emu container's IP
$ export VIM_EMU_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' vim-emu)

# connect OSM to emulated VIM
$ osm vim-create --name emu-vim1 --user username --password password --auth_url http://$VIM_EMU_IP:6001/v2.0 --tenant tenantName --account_type openstack

# list vims
$ osm vim-list
+----------+--------------------------------------+
| vim name | uuid                                 |
+----------+--------------------------------------+
| emu-vim1 | a8175948-efcf-11e7-94ad-00163eba993f |
+----------+--------------------------------------+
```

## On-board example service

```sh
# VNFs
$ osm upload-package ping.tar.gz
$ osm upload-package pong.tar.gz

# NS
$ osm upload-package pingpong_nsd.tar.gz

# You can now check OSM's Launchpad to see the VNFs and NS in the catalog. Or:
$ osm vnfd-list
+-----------+------+
| vnfd name | id   |
+-----------+------+
| ping      | ping |
| pong      | pong |
+-----------+------+

$ osm nsd-list
+----------+----------+
| nsd name | id       |
+----------+----------+
| pingpong | pingpong |
+----------+----------+
```

## Instantiate example service

```sh
$ osm ns-create --nsd_name pingpong --ns_name test-nsi --vim_account emu-vim1
```

## Check service instance

Use the OSM client to check the service instance's state:
```sh
$ osm vnf-list
+----------------------------+--------------------------------------+--------------------+-------------------+
| vnf name                   | id                                   | operational status | config status     |
+----------------------------+--------------------------------------+--------------------+-------------------+
| default__test-nsi__pong__2 | 788fd000-c0f2-4921-ace1-54646b804c9d | pre-init           | config-not-needed |
| default__test-nsi__ping__1 | 8cf2db31-58de-4454-85af-864a968846cf | pre-init           | config-not-needed |
+----------------------------+--------------------------------------+--------------------+-------------------+

$ osm ns-list
+------------------+--------------------------------------+--------------------+---------------+
| ns instance name | id                                   | operational status | config status |
+------------------+--------------------------------------+--------------------+---------------+
| test-nsi         | ea746088-efcf-11e7-bef9-005056b887c5 | running            | configured    |
+------------------+--------------------------------------+--------------------+---------------+
```

Use vim-emu client to check for the deployed container-based VNFs:
```sh
$ docker exec vim-emu vim-emu compute list
+--------------+----------------------------+---------------+------------------+-------------------------+
| Datacenter   | Container                  | Image         | Interface list   | Datacenter interfaces   |
+==============+============================+===============+==================+=========================+
| dc1          | dc1_test-nsi.ping.1.ubuntu | ubuntu:trusty | ping0-0          | dc1.s1-eth2             |
+--------------+----------------------------+---------------+------------------+-------------------------+
| dc1          | dc1_test-nsi.pong.2.ubuntu | ubuntu:trusty | pong0-0          | dc1.s1-eth3             |
+--------------+----------------------------+---------------+------------------+-------------------------+

```

Interact with the deployed service:
```sh
# connect to ping VNF container:
$ docker exec -it mn.dc1_test-nsi.ping.1.ubuntu /bin/bash

# show network config
root@dc1_test-nsi:/# ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:03
          inet addr:172.17.0.3  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:648 (648.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

ping0-0   Link encap:Ethernet  HWaddr 4a:57:93:a0:d4:9d
          inet addr:192.168.100.3  Bcast:192.168.100.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
          
# ping the pong VNF over the attached management network
root@dc1_test-nsi:/# ping -c2 192.168.100.4
PING 192.168.100.4 (192.168.100.4) 56(84) bytes of data.
64 bytes from 192.168.100.4: icmp_seq=1 ttl=64 time=0.070 ms
64 bytes from 192.168.100.4: icmp_seq=2 ttl=64 time=0.048 ms

--- 192.168.100.4 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.048/0.059/0.070/0.011 ms
```

## Terminate service
```sh
osm ns-delete test-nsi
```

# Helper Notes

Stop vim-emu:
```sh
$ docker stop vim-emu
```

Check if vim-emu is available from RO:

```sh
$ lxc exec RO -- /bin/bash
# set openstack env.:
export OS_USERNAME=username
export OS_PASSWORD=password
export OS_TENANT_NAME=tenantName
export OS_TENANT_ID=fc394f2ab2df4114bde39905f800dc57
export OS_AUTH_URL=http://172.17.0.2:6001/v2.0  # Attention: replace with vim-emu Docker container IP
export OS_IDENTITY_API_VERSION=2
export OS_REGION_NAME=RegionOne

# check connectivity
$ openstack flavor list
$ openstack network list
$ openstack image list
```
