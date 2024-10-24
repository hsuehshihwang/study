# openWRT study with QEMU and name space

## Introduction

> Use the simple way to use QEMU utilities to test the openWRT image and separate the networking space. 

## Quick Start

### Step 1 - Create name spaces and VETH peer interfaces

> - ns_openwrt: for qemu image
> - ns_openwrt_wan: for wan networking
> - veth pairs for WAN: veth_wan, veth_qemu_wan
> - veth pairs for LAN: veth_lan, veth_qemu_lan

```bash
# Create name space
sudo ip netns add ns_openwrt_wan
sudo ip netns add ns_openwrt
sudo ip netns list
```

### Step 2 - Set up networking interfaces

> - veth pairs for WAN: veth_wan, veth_qemu_wan
> - veth pairs for LAN: veth_lan, veth_qemu_lan
> - Bridge intefaces: br_qemu_wan, br_qemu_lan
> - TAP interfaces: tap_qemu_wan, tap_qemu_lan

```bash
# Create VETH pairs
sudo ip link add veth_wan type veth peer name veth_qemu_wan
sudo ip link add veth_lan type veth peer name veth_qemu_lan
sudo ip link set veth_wan netns ns_openwrt_wan
sudo ip link set veth_qemu_wan netns ns_openwrt
sudo ip link set veth_qemu_lan netns ns_openwrt
# Set up WAN VETH interfaces IP
sudo ip netns exec ns_openwrt_wan ip link set veth_wan up
sudo ip netns exec ns_openwrt_wan ip addr add 192.168.100.1/24 dev veth_wan
# Set up LAN VETH interfaces IP
sudo ip link set veth_lan up
sudo ip addr add 192.168.1.125/24 dev veth_lan
# Up the LAN / WAN VETH intefaces for QEMU image name space
sudo ip netns exec ns_openwrt ip link set veth_qemu_wan up
sudo ip netns exec ns_openwrt ip link set veth_qemu_lan up
# Create TAP interfaces for QEMU and bridge LAN / WAN VETH and TAP interfaces
sudo ip netns exec ns_openwrt ip tuntap add mode tap tap_qemu_wan
sudo ip netns exec ns_openwrt ip link set tap_qemu_wan up
sudo ip netns exec ns_openwrt ip tuntap add mode tap tap_qemu_lan
sudo ip netns exec ns_openwrt ip link set tap_qemu_lan up
sudo ip netns exec ns_openwrt brctl addbr br_qemu_wan
sudo ip netns exec ns_openwrt brctl addif br_qemu_wan tap_qemu_wan
sudo ip netns exec ns_openwrt brctl addif br_qemu_wan veth_qemu_wan
sudo ip netns exec ns_openwrt brctl addbr br_qemu_lan
sudo ip netns exec ns_openwrt brctl addif br_qemu_lan tap_qemu_lan
sudo ip netns exec ns_openwrt brctl addif br_qemu_lan veth_qemu_lan
sudo ip netns exec ns_openwrt ip link set br_qemu_wan up
sudo ip netns exec ns_openwrt ip link set br_qemu_lan up

```

### Step 3 - Enter name space and run QEMU

#### Enter QEMU name space

```bash
# Enter QEMU name space
sudo ip netns exec ns_openwrt bash
ip netns identify
```

#### Download image and unpack image

```bash
# Download image and unpack image
wget https://archive.openwrt.org/releases/21.02.0/targets/x86/64/openwrt-21.02.0-x86-64-generic-ext4-combined.img.gz
gunzip openwrt-21.02.0-x86-64-generic-ext4-combined.img.gz
```

#### Run QEMU

```bash
# Run QEMU with image
qemu-system-x86_64 -m 512M \
    -drive file=openwrt-21.02.0-x86-64-generic-ext4-combined.img,format=raw \
    -netdev tap,id=net0,ifname=tap_qemu_lan,script=no,downscript=no \
    -device virtio-net,netdev=net0 \
    -netdev tap,id=net1,ifname=tap_qemu_wan,script=no,downscript=no \
    -device virtio-net,netdev=net1 \
    -nographic
```

#### Shutdown QEMU

##### poweroff

```bash
root@OpenWrt:/# poweroff
```

##### shortcut key

```bash
ctrl + A, then X
```

### Step 4 - Host side test

> Host side you have a networking inteface (veth_lan) and static IP 192.168.1.125

#### PING

```bash
ping 192.168.1.1
```

#### Remote side to redirect the SSH and HTTP

```bash
# Remote ssh client to open 22222 to connect to openWRT 22(SSH)
ssh -f -N -L 22222:192.168.1.1:22 root@${remote_server}
# Test SSH
ssh -p 22222 root@localhost 

# openWRT GUI
ssh -f -N -L 28080:192.168.1.1:80 root@${remote_server}
# Test HTTP
http://localhost:28080/
```






