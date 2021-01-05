---
layout: post
title: "Configuring Raspberry PI as an OpenFlow switch"
date: 2019-02-01 23:29:00 +0000
tags: sdn raspberrypi openflow
description: "Step by step instructions on setting up a Raspberry PI as an OpenFlow switch"
---
## Follow the following steps to setup the OpenFlow switch
1. Download openvswitch
```shell
wget http://openvswitch.org/releases/openvswitch-2.5.2.tar.gz
```

1. Unpack archive
```shell
tar -xvzf openvswitch-2.5.2.tar.gz
```

1.  Install following dependancies
```shell
apt-get install python-simplejson python-qt4 libssl-dev python-twisted-conch automake autoconf gcc uml-utilities libtool build-essential pkg-config
apt-get install linux-headers-3.10-3-rpi
```

1. Make the switch (Navigate to `openvswitch-2.5.2` and enter the following commands)
```shell
./configure --with-linux=/lib/modules/3.10-3-rpi/build
make
make install
```

1. Turn on openvswitch module
```shell
cd openvswitch-2.5.2/datapath/linux
modprobe openvswitch
```

1. Create `ovs_script.sh` with the following code<br>
```shell
#!/bin/bash
ovsdb-server --remote=punix:/usr/local/var/run/openvswitch/db.sock \
    --remote=db:Open_vSwitch,Open_vSwitch,manager_options \
    --private-key=db:Open_vSwitch,SSL,private_key \
    --certificate=db:Open_vSwitch,SSL,certificate \
    --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert \
    --pidfile –detach
ovs-vsctl --no-wait init
ovs-vswitchd --pidfile –detach
ovs-vsctl show
```

1. Create a file for the database, which will contain the details of the switch
```shell
touch /usr/local/etc/ovs-vswitchd.conf
```

1. Create the following directory
```shell
mkdir -p /usr/local/etc/openvswitch
```

1. Populate the database, which will be used by the ovswitch
```shell
./openvswitch-2.5.2/ovsdb/ovsdb-tool create /usr/local/etc/openvswitch/conf.db openvswitch-2.5.2/vswitchd/vswitch.ovsschema
```

1. Run `ovs_script.sh`

1. Add a new bridge
```shell
ovs-vsctl add-br br0
```

1. Bind the ports to the newly added bridge
```shell
ifconfig eth1 0 up
ifconfig eth2 0 up
ifconfig eth3 0 up
ifconfig eth4 0 up
```

1. Set the interfaces up
```shell
ifconfig eth1 0 up
ifconfig eth2 0 up
ifconfig eth3 0 up
ifconfig eth4 0 up
```

1. Connect the switch to an external controller
```shell
ovs-vsctl set-controller br0 tcp:20.0.0.7:6634
```

## Configuring the switch to initialize as an OpenFlow switch at startup

- Add the following bash script to the location of `openvswitch-2.5.2` and rename it to `main_script.sh`
```shell
#!/bin/bash
cd openvswitch-2.5.2/datapath/linux
modprobe openvswitch
cd ..
cd ..
cd ..
./ovs_script.sh
ifconfig eth1 0 up
ifconfig eth2 0 up
ifconfig eth3 0 up
ifconfig eth4 0 up
ifconfig br0 0 up
```

- Then add the following line at the end of `.bashrc`
```shell
sudo sh [location of main_script.sh]/main_script.sh
```

## Extracting the hex value of DPID in a convenient way

- add the following file to an arbitrary location and rename it to `dpid.sh`
```shell
#!/bin/bash
var=$(cat /sys/class/net/br0/address)
output=$(echo "$var" | tr --delete :)
echo "DPID : " $((0x${output}))
```

- Then run the above script using the following command
```shell
sh dpid.sh
```