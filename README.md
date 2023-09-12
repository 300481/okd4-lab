# OKD4 Lab Files

## Virtualization Host

Fedora CoreOS + KVM + Cockpit + Bridged Network (virtual networks IP range is part of the physical network)

64GB RAM

8 CPU Cores

2x 1TB SSD

## Network

| Property | Value          |
|----------|----------------|
| Subnet   | 192.168.0.0/24 |
| Gateway  | 192.168.0.1    |
| DNS      | 192.168.0.1    |

## Bastion Host

### OS

Fedora Server

### Sizing

8 GB RAM

4 vCPUs

50 GB Storage

### Packages

- *bind*
- *nginx*
- *git*
- *podman*

### Network Setup

| Property | Value          |
|----------|----------------|
| Subnet   | 192.168.0.0/24 |
| Gateway  | 192.168.0.1    |
| DNS      | 192.168.0.1    |
| IP       | 192.168.0.200  |

## Bootstrap Node

### Sizing

32 GB RAM

4 vCPUs

120 GB Storage

### OS

Fedora CoreOS

### Network Setup

| Property | Value           |
|----------|-----------------|
| Subnet   | 192.168.0.0/24  |
| Gateway  | 192.168.0.1     |
| DNS      | *192.168.0.200* |
| IP       | 192.168.0.151   |

## Master Node

### Sizing

32 GB RAM

8 vCPUs

120 GB Storage

### OS

Fedora CoreOS

### Network Setup

| Property | Value           |
|----------|-----------------|
| Subnet   | 192.168.0.0/24  |
| Gateway  | 192.168.0.1     |
| DNS      | *192.168.0.200* |
| IP       | 192.168.0.152   |

## preparation

### clone git repo

```bash
git clone https://github.com/300481/okd4-lab.git
cd okd4-lab
```

### copy DNS Server config

```bash
sudo cp -rv dns/etc/* /etc/
```

### download OpenShift CLI and Installer

Get the OKD version from the [git repository](https://github.com/okd-project/okd/releases)

```bash
OC_VERSION=4.13.0-0.okd-2023-07-23-051208

mkdir cli
cd cli

wget https://github.com/okd-project/okd/releases/download/${OC_VERSION}/openshift-client-linux-${OC_VERSION}.tar.gz
wget https://github.com/okd-project/okd/releases/download/${OC_VERSION}/openshift-install-linux-${OC_VERSION}.tar.gz

tar xvzf openshift-client-linux-${OC_VERSION}.tar.gz
tar xvzf openshift-install-linux-${OC_VERSION}.tar.gz

sudo mv oc kubectl openshift-install /usr/local/bin/
cd ..
rm -rf cli
```

### create okd4 directory for webserver

```bash
sudo mkdir -p /usr/share/nginx/html/okd4
```

### download and prepare the Fedora CoreOS image

```bash
alias coreos-installer='podman run --privileged --rm \
        -v /dev:/dev \
        -v /run/udev:/run/udev \
        -v ${PWD}:/data \
        -w /data quay.io/coreos/coreos-installer:release'

wget -O fcos.iso $(openshift-install coreos print-stream-json | grep location | grep x86 | grep iso | cut -d\" -f4)

cp fcos.iso fcos-bootstrap.iso
cp fcos.iso fcos-master.iso

coreos-installer iso kargs modify -a "ip=192.168.0.151::192.168.0.1:255.255.255.0::enp1s0:none nameserver=192.168.0.200 console=ttyS0" fcos-bootstrap.iso
coreos-installer iso kargs modify -a "ip=192.168.0.152::192.168.0.1:255.255.255.0::enp1s0:none nameserver=192.168.0.200 console=ttyS0" fcos-master.iso

coreos-installer iso kargs modify -r ignition.platform.id=metal=qemu fcos-bootstrap.iso
coreos-installer iso kargs modify -r ignition.platform.id=metal=qemu fcos-master.iso

sudo mv fcos-*.iso /usr/share/nginx/html/okd4/
```

### create ignition files for bootstrap and master node

```bash
alias butane='podman run --rm --interactive       \
              --security-opt label=disable        \
              --volume ${PWD}:/pwd --workdir /pwd \
              quay.io/coreos/butane:release       \
              --pretty --strict'

cd okd

rmdir -rf config
mkdir config

cp install-config.yaml config/
openshift-install create manifests --dir config/
openshift-install create ignition-configs --dir config/

mv config/bootstrap.ign ignition/
mv config/master.ign ignition/
cd ignition
butane bootstrap-config.yaml -d . > bootstrap-merged.ign
butane master-config.yaml -d . > master-merged.ign

sudo mv *merged* /usr/share/nginx/html/okd4/

cd ..
```

### create bootstrap and master VMs

Boot them with the following ISO image

| Node      | Image                                        |
|-----------|----------------------------------------------|
| Bootstrap | http://192.168.0.200/okd4/fcos-bootstrap.iso |
| Master    | http://192.168.0.200/okd4/fcos-master.iso    |

### install the bootstrap and master VMs

#### Installation Commands

| Node      | Command                                      |
|-----------|----------------------------------------------|
| Bootstrap | ```sudo coreos-installer install /dev/vda -n --insecure-ignition --ignition-url=http://192.168.0.200/okd4/bootstrap-merged.ign ; reboot``` |
| Master    | ```sudo coreos-installer install /dev/vda -n --insecure-ignition --ignition-url=http://192.168.0.200/okd4/master-merged.ign ; reboot``` |

#### Wait for Bootstrap Completion

```bash
openshift-install wait-for bootstrap-complete --dir config/
```

#### Adjust DNS

Comment entries of api.okd.lab + api-int.okd.lab for Bootstrap Node

Count up the serial +1

*/etc/named/zones/db.lab*

```
@       IN      SOA     ns.lab. admin.lab. (
        6          ; Serial
        604800     ; Refresh
        86400      ; Retry
        2419200    ; Expire
        604800 )   ; Negative Cache TTL
;
; name servers - NS records
    IN      NS     ns.lab.

; name servers - A records
ns.lab.            IN      A      192.168.0.200
bootstrap.okd.lab. IN      A      192.168.0.151 ; Bootstrap
*.apps.okd.lab.    IN      A      192.168.0.152 ; Master
api.okd.lab.       IN      A      192.168.0.152 ; Master
; api.okd.lab.       IN      A      192.168.0.151 ; Bootstrap
api-int.okd.lab.   IN      A      192.168.0.152 ; Master
; api-int.okd.lab.   IN      A      192.168.0.151 ; Bootstrap
master.okd.lab.    IN      A      192.168.0.152 ; Master
```

Restart *named*

```bash
sudo systemctl restart named
```

#### Delete Bootstrap Node

You can do this safely, it's not needed anymore.

#### Wait for Install Completion

```bash
openshift-install wait-for install-complete --dir config/
```

Here we go!!!