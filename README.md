# OpenShift Agent-Based Installation
The Installation below is based off of OpenShift 4.19.7. An Agent-based installation was chosen to quickly intergrate NVIDIA GPU's and external Dell PowerScale storage for PV creation


##prerequisite
DNS "A" and "PTR" or local host files entries made to support the installation

## Ubuntu Configuration

Ubuntu version 24.04.3

Install Ubuntu on two servers. Configure with IP's from the OpenShift Machine Network subnet. 

After installation run the following commands:
```bash
sudo apt update -y; sudo upgrade -y

sudo apt install haproxy keepalived -y

sudo apt update -y

sudo ufw allow 22; sudo ufw allow 8404; sudo ufw allow 6443; sudo ufw allow 22623; sudo ufw allow 443

sudo ufw enable
sudo systemctl enable ufw

```
## HAProxy & Keepalived

### HAProxy-01 Example Configuration

Replace /etc/haproxy/haproxy.cfg with content below:

```bash

#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   https://www.haproxy.org/download/1.8/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local0 info
     


    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

    # utilize system-wide crypto-policies
    #ssl-default-bind-ciphers PROFILE=SYSTEM
    #ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    
    mode                    tcp
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------

frontend stats
   mode http
   bind *:8404
   stats enable
   stats refresh 10s
   stats uri /stats
   stats show-modules
   stats realm Haproxy\ Statistics
   stats show-legends
   stats show-node
   stats admin if TRUE

frontend api
   bind *:6443 
   mode tcp 
   default_backend controlplaneapi

frontend apiinternal
   mode tcp
   bind *:22623
   default_backend controlplaneapiinternal

frontend secure
   bind *:443 
   mode tcp
   default_backend secure

frontend insecure
   mode tcp
   bind *:80
   default_backend insecure

#---------------------------------------------------------------------
# static backend
#---------------------------------------------------------------------

backend controlplaneapi
   mode tcp
   balance roundrobin
   server controller-01 10.151.87.20:6443 check
   server controller-02 10.151.87.21:6443 check
   server controller-03 10.151.87.22:6443 check


backend controlplaneapiinternal
   mode tcp
   balance roundrobin
   server controller-01 10.151.87.20:22623 check
   server controller-02 10.151.87.21:22623 check
   server controller-03 10.151.87.22:22623 check

backend secure
   mode tcp
   balance roundrobin
   server work-01 10.151.87.23:443 check
   server work-02 10.151.87.24:443 check
    
backend insecure
   mode tcp
   balance roundrobin
   server work-01 10.151.87.23:80 check
   server work-02 10.151.87.24:80 check

```

## Restart service

```bash

sudo systemctl restart haproxy.service

```

### HAProxy-01 Keepalived Example Configuration

Replace /etc/keepalived/keepalived.cfg with content below:

```bash
# Global definitions for the keepalived instance
global_defs {
    router_id LBR-MASTER  # A unique identifier for this node
    # notification_email { ... } # Optional email notifications
}

# Health check script to verify the local load balancer (e.g., HAProxy) is running
vrrp_script check_haproxy {
    script "killall -0 haproxy" # Checks if haproxy process is running
    interval 2                 # Run every 2 seconds
    weight 20                  # Add weight to priority if script passes
    fall 2                     # Fail after 2 consecutive failures
    rise 2                     # Pass after 2 consecutive successes
}

# VRRP Instance for the OpenShift API VIP
vrrp_instance VI_OS_API {
    state MASTER             # Initial state
    interface ens33          # The network interface connected to the OCP network
    virtual_router_id 51     # Unique VRRP ID for the API instance (1-255)
    priority 200             # Priority (higher wins election)
    advert_int 1             # Advertisement interval in seconds
    authentication {
        auth_type PASS       # Simple password authentication
        auth_pass apipass    # Must match on all load balancers
    }
    virtual_ipaddress {
        10.151.87.9/24 dev ens33       # The OpenShift API VIP (api.<cluster-name>.<base-domain>)
        10.151.87.10/24 dev ens33     # The OpenShift Ingress VIP (*.apps<cluster-name>.<base-domain>)
    }
    track_script {
        check_haproxy        # Track the health script
    }
}

```

## Restart service

```bash

sudo systemctl restart keeplived.service

```

### HAProxy-02 Example Configuration

Replace /etc/haproxy/haproxy.cfg with content below:

```bash

#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   https://www.haproxy.org/download/1.8/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

    # utilize system-wide crypto-policies
    #ssl-default-bind-ciphers PROFILE=SYSTEM
    #ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    tcp
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------

frontend stats
  mode http
  bind *:8404
  stats enable
  stats refresh 10s
  stats uri /stats
  stats show-modules
  stats admin if TRUE

frontend api
    bind *:6443 
    mode tcp
    default_backend controlplaneapi

frontend apiinternal
    bind *:22623
    mode tcp
    default_backend controlplaneapiinternal

frontend secure
    bind *:443 
    mode tcp
    default_backend secure

frontend insecure
    bind *:80
    mode tcp
    default_backend insecure

#---------------------------------------------------------------------
# static backend
#---------------------------------------------------------------------

backend controlplaneapi
    mode tcp
    balance roundrobin
    server controller-01 10.151.87.20:6443 check
    server controller-02 10.151.87.21:6443 check
    server controller-03 10.151.87.22:6443 check


backend controlplaneapiinternal
    mode tcp
    balance roundrobin
    server controller-01 10.151.87.20:22623 check
    server controller-02 10.151.87.21:22623 check
    server controller-03 10.151.87.22:22623 check

backend secure
    mode tcp
    balance roundrobin
    server work-01 10.151.87.23:443 check
    server work-02 10.151.87.24:443 check
    
backend insecure
    mode tcp
    balance roundrobin
    server work-01 10.151.87.23:80 check
    server work-02 10.151.87.24:80 check
   
```
## Restart service

```bash

sudo systemctl restart haproxy.service

```

### HAProxy-02 Keepalivd Example Configuration

Replace /etc/keepalived/keepalived.cfg with content below:

```bash
# Global definitions for the keepalived instance
global_defs {
    router_id LBR-MASTER  # A unique identifier for this node
    # notification_email { ... } # Optional email notifications
}

# Health check script to verify the local load balancer (e.g., HAProxy) is running
vrrp_script check_haproxy {
    script "killall -0 haproxy" # Checks if haproxy process is running
    interval 2                 # Run every 2 seconds
    weight 20                  # Add weight to priority if script passes
    fall 2                     # Fail after 2 consecutive failures
    rise 2                     # Pass after 2 consecutive successes
}

# VRRP Instance for the OpenShift API VIP
vrrp_instance VI_OS_API {
    state BACKUP             # Initial state
    interface ens33          # The network interface connected to the OCP network
    virtual_router_id 51     # Unique VRRP ID for the API instance (1-255)
    priority 100             # Priority (higher wins election)
    advert_int 1             # Advertisement interval in seconds
    authentication {
        auth_type PASS       # Simple password authentication
        auth_pass apipass    # Must match on all load balancers
    }
    virtual_ipaddress {
        10.151.87.9/24 dev ens33       # The OpenShift API VIP (api.<cluster-name>.<base-domain>)
        10.151.87.10/24 dev ens33     # The OpenShift Ingress VIP (*.apps<cluster-name>.<base-domain>)  
    }
    track_script {
        check_haproxy        # Track the health script
    }
}

```
## Restart service

```bash

sudo systemctl restart keeplived.service

```

## Agent-config.yaml Example
```yaml
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: workcluster
rendezvousIP: 10.151.87.20
hosts:
  - hostname: controller-01
    role: master
    interfaces:
      - name: ens32
        macAddress: "00:50:56:91:15:73"
    networkConfig:
      interfaces:
        - name: ens32
          type: ethernet
          state: up
          mtu: "9000"
          ipv4:
            enabled: true
            address:
              - ip: 10.151.87.20
                prefix-length: 24
      dns-resolver:
        config:
          server:
            - 10.151.87.251
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.151.87.1
            next-hop-interface: ens32

  - hostname: controller-02
    role: master
    interfaces:
      - name: ens32
        macAddress: "00:50:56:91:1d:2b"
    networkConfig:
      interfaces:
        - name: ens32
          type: ethernet
          state: up
          mtu: "9000"
          ipv4:
            enabled: true
            address:
              - ip: 10.151.87.21
                prefix-length: 24
      dns-resolver:
        config:
          server:
            - 10.151.87.251
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.151.87.1
            next-hop-interface: ens32

  - hostname: controller-03
    role: master
    interfaces:
      - name: ens32
        macAddress: "00:50:56:91:70:a3"
    networkConfig:
      interfaces:
        - name: ens32
          type: ethernet
          state: up
          mtu: "9000"
          ipv4:
            enabled: true
            address:
              - ip: 10.151.87.22
                prefix-length: 24
      dns-resolver:
        config:
          server:
            - 10.151.87.251
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.151.87.1
            next-hop-interface: ens32

  - hostname: work-01
    role: worker
    interfaces:
      - name: ens32
        macAddress: "00:50:56:91:53:c2"
    networkConfig:
      interfaces:
        - name: ens32
          type: ethernet
          state: up
          mtu: "9000"
          ipv4:
            enabled: true
            address:
              - ip: 10.151.87.23
                prefix-length: 24
      dns-resolver:
        config:
          server:
            - 10.151.87.251
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.151.87.1
            next-hop-interface: ens32

  - hostname: work-02
    role: worker
    interfaces:
      - name: ens32
        macAddress: "00:50:56:91:39:8d"
    networkConfig:
      interfaces:
        - name: ens32
          type: ethernet
          state: up
          mtu: "9000"
          ipv4:
            enabled: true
            address:
              - ip: 10.151.87.24
                prefix-length: 24
      dns-resolver:
        config:
          server:
            - 10.151.87.251
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.151.87.1
            next-hop-interface: ens32

```
## Install-config.yaml Example
```yaml
apiVersion: v1
baseDomain: sterling.xyz
metadata:
  name: workcluster
compute:
  - name: worker
    replicas: 2
controlPlane:
  name: master
  replicas: 3
networking:
  machineNetwork:
  - cidr: 10.151.87.0/24
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  baremetal:
    apiVIPs: 
      - 10.151.87.99
    ingressVIPs: 
      - 10.151.87.100
pullSecret: 'your pull secret here'
sshKey: 'your ssh public key here'
```
## Ubuntu Desktop Live CD

From the lights-out controller on each Master and Worker node, mount and boot into ubuntu-22.04.5-desktop-amd64.iso. We will run the from memory rather than install onto the physical hard drives. With the OS loaded we can gather the MAC addresses, interface names, and device id's need for the OpenShift installation

Open a console window on each server install ssh and set default passwd

```bash

sudo apt install ssh-server -y; sudo passwd ubuntu

```

```bash

ip addr

```

Establish ssh session to each server and run the following commands to identify spare disks that can be utilized for ephemeral storage

```bash

lsblk

```

Identify spare disks serial numbers which will be used in butain file

```bash

ls -l /dev/disk/by-id/

#udevadm info --query=all --name=/dev/sda | grep ID_SERIAL

```

Appropriately update the Agent-config.yaml with the interface names and MAC addresses previously captured

# Create Butane files

## 99-master-m0-storage.bu Example
```bash

variant: openshift
version: 4.19.0
metadata:
  name: 99-master-m0
  labels:
    machineconfiguration.openshift.io/role: master
    # Add a specific label for this node
    specific-node: controller-01
storage:
  # ... (filesystems definition remains the same)
  filesystems:
    - path: /var/lib/containers
      device: /dev/mapper/vg_containers-lv_containers
      format: xfs
      label: var-lib-cont
      wipe_filesystem: true
systemd:
  units:
    - name: setup-containers-lvm.service
      enabled: true
      contents: |
        [Unit]
        Description=Set up LVM for /var/lib/containers (master-0)
        Before=crio.service kubelet.service
        [Service]
        Type=oneshot
        RemainAfterExit=yes
        # Use the specific IDs for master-0
        ExecStart=/usr/sbin/pvcreate /dev/disk/by-id/scsi-36000c296c4cf076c02aa4b27c9af3448 /dev/disk/by-id/scsi-36000c29d513627950d35e5ed6b1487d1 /dev/disk/by-id/scsi-36000c2904e28e740c426ee7860aec6aa
        ExecStart=/usr/sbin/vgcreate vg_containers /dev/disk/by-id/scsi-36000c296c4cf076c02aa4b27c9af3448 /dev/disk/by-id/scsi-36000c29d513627950d35e5ed6b1487d1 /dev/disk/by-id/scsi-36000c2904e28e740c426ee7860aec6aa
        ExecStart=/usr/sbin/lvcreate -l 100%FREE -n lv_containers vg_containers
        ExecStart=/usr/sbin/mkfs.xfs /dev/mapper/vg_containers-lv_containers
        [Install]
        WantedBy=multi-user.target

```

## 99-master-m1-storage.bu Example
```bash

variant: openshift
version: 4.19.0
metadata:
  name: 99-master-m1
  labels:
    machineconfiguration.openshift.io/role: master
    # Add a specific label for this node
    specific-node: controller-02
storage:
  # ... (filesystems definition remains the same)
  filesystems:
    - path: /var/lib/containers
      device: /dev/mapper/vg_containers-lv_containers
      format: xfs
      label: var-lib-cont
      wipe_filesystem: true
systemd:
  units:
    - name: setup-lvm.service
      enabled: true
      contents: |
        [Unit]
        Description=Set up LVM for /var/lib/containers (master-0)
        Before=crio.service kubelet.service
        [Service]
        Type=oneshot
        RemainAfterExit=yes
        # Use the specific IDs for master-m1
        ExecStart=/usr/sbin/pvcreate /dev/disk/by-id/scsi-36000c294def6476b58dfe1ab54b4b1de /dev/disk/by-id/scsi-36000c2980acad421860c1eec67118024 /dev/disk/by-id/scsi-36000c29366f6507bb3d765c496caf368
        ExecStart=/usr/sbin/vgcreate vg_containers /dev/disk/by-id/scsi-36000c294def6476b58dfe1ab54b4b1de /dev/disk/by-id/scsi-36000c2980acad421860c1eec67118024 /dev/disk/by-id/scsi-36000c29366f6507bb3d765c496caf368
        ExecStart=/usr/sbin/lvcreate -l 100%FREE -n lv_containers vg_containers
        ExecStart=/usr/sbin/mkfs.xfs /dev/mapper/vg_containers-lv_containers
        [Install]
        WantedBy=multi-user.target
```

## 99-master-m2-storage.bu Example
```bash

variant: openshift
version: 4.19.0
metadata:
  name: 99-master-m2
  labels:
    machineconfiguration.openshift.io/role: master
    # Add a specific label for this node
    specific-node: controller-03
storage:
  # ... (filesystems definition remains the same)
  filesystems:
    - path: /var/lib/containers
      device: /dev/mapper/vg_containers-lv_containers
      format: xfs
      label: var-lib-cont
      wipe_filesystem: true
systemd:
  units:
    - name: setup-lvm.service
      enabled: true
      contents: |
        [Unit]
        Description=Set up LVM for /var/lib/containers (master-0)
        Before=crio.service kubelet.service
        [Service]
        Type=oneshot
        RemainAfterExit=yes
        # Use the specific IDs for master-m2
        ExecStart=/usr/sbin/pvcreate /dev/disk/by-id/scsi-36000c294def6476b58dfe1ab54b4b1de /dev/disk/by-id/scsi-36000c2901de05db73a89bb7bf6d7998d /dev/disk/by-id/scsi-36000c29203ef255d524beace20bbbd15
        ExecStart=/usr/sbin/vgcreate vg_containers /dev/disk/by-id/scsi-36000c294def6476b58dfe1ab54b4b1de /dev/disk/by-id/scsi-36000c2901de05db73a89bb7bf6d7998d /dev/disk/by-id/scsi-36000c29203ef255d524beace20bbbd15
        ExecStart=/usr/sbin/lvcreate -l 100%FREE -n lv_containers vg_containers
        ExecStart=/usr/sbin/mkfs.xfs /dev/mapper/vg_containers-lv_containers
        [Install]
        WantedBy=multi-user.target

```

## 99-master-w0-storage.bu Example
```bash

variant: openshift
version: 4.19.0
metadata:
  name: 99-worker-w0
  labels:
    machineconfiguration.openshift.io/role: master
    # Add a specific label for this node
    specific-node: work-01
storage:
  # ... (filesystems definition remains the same)
  filesystems:
    - path: /var/lib/containers
      device: /dev/mapper/vg_containers-lv_containers
      format: xfs
      label: var-lib-cont
      wipe_filesystem: true
systemd:
  units:
    - name: setup-lvm.service
      enabled: true
      contents: |
        [Unit]
        Description=Set up LVM for /var/lib/containers (master-0)
        Before=crio.service kubelet.service
        [Service]
        Type=oneshot
        RemainAfterExit=yes
        # Use the specific IDs for worker-w0
        ExecStart=/usr/sbin/pvcreate /dev/disk/by-id/scsi-36000c291dc9f1add19674637b4a94bd9 /dev/disk/by-id/scsi-36000c295737ab115dcf88a3668e6d923 /dev/disk/by-id/scsi-36000c29a39e3ce29114ba833a54d1190
        ExecStart=/usr/sbin/vgcreate vg_containers /dev/disk/by-id/scsi-36000c291dc9f1add19674637b4a94bd9 /dev/disk/by-id/scsi-36000c295737ab115dcf88a3668e6d923 /dev/disk/by-id/scsi-36000c29a39e3ce29114ba833a54d1190
        ExecStart=/usr/sbin/lvcreate -l 100%FREE -n lv_containers vg_containers
        ExecStart=/usr/sbin/mkfs.xfs /dev/mapper/vg_containers-lv_containers
        [Install]
        WantedBy=multi-user.target

```

## 99-master-w1-storage.bu Example
```bash

variant: openshift
version: 4.19.0
metadata:
  name: 99-worker-w1
  labels:
    machineconfiguration.openshift.io/role: master
    # Add a specific label for this node
    specific-node: work-02
storage:
  # ... (filesystems definition remains the same)
  filesystems:
    - path: /var/lib/containers
      device: /dev/mapper/vg_containers-lv_containers
      format: xfs
      label: var-lib-cont
      wipe_filesystem: true
systemd:
  units:
    - name: setup-lvm.service
      enabled: true
      contents: |
        [Unit]
        Description=Set up LVM for /var/lib/containers (master-0)
        Before=crio.service kubelet.service
        [Service]
        Type=oneshot
        RemainAfterExit=yes
        # Use the specific IDs for worker-w1
        ExecStart=/usr/sbin/pvcreate /dev/disk/by-id/scsi-36000c295abdb746293cf8e7c5562e978 /dev/disk/by-id/scsi-36000c291e6e5fb6f64f8036b13b4a11f /dev/disk/by-id/scsi-36000c293050e4b5f9708190585238465
        ExecStart=/usr/sbin/vgcreate vg_containers /dev/disk/by-id/scsi-36000c295abdb746293cf8e7c5562e978 /dev/disk/by-id/scsi-36000c291e6e5fb6f64f8036b13b4a11f /dev/disk/by-id/scsi-36000c293050e4b5f9708190585238465
        ExecStart=/usr/sbin/lvcreate -l 100%FREE -n lv_containers vg_containers
        ExecStart=/usr/sbin/mkfs.xfs /dev/mapper/vg_containers-lv_containers
        [Install]
        WantedBy=multi-user.target

```

## Create Agent ISO

The Agent ISO can be created on Windows, MacOS, or Linux. I've chosen RHEL Developer version which is free for illustration purposes. I chose RHEL because getting NMState installed is more straight forward than using other distro's. Use the following URL to identify and download the specific version of OpenShift tools required for your installation:

https://mirror.openshift.com/pub/openshift-v4/clients/ocp/

Requirements:<br />
openshift-client-linux-4.19.17.tar.gz<br />
openshift-install-linux-4.19.17.tar.gz

```bash
cd  /home/sterling/Downloads
curl -O https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.19.17/openshift-client-linux-4.19.17.tar.gz
curl -O https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.19.17/openshift-install-linux-4.19.17.tar.gz

tar -xvzf /home/sterling/Downloads/openshift-client-linux-4.19.17.tar.gz
tar -xvzf /home/sterling/Downloads/openshift-install-linux-4.19.17.tar.gz

sudo mv /home/sterling/Downloads/oc kubectl openshift-install /usr/sbin
mkdir /home/sterling/install
mkdir /home/sterling/install/original

sudo touch /home/sterling/install/original/agent-config.yaml
sudo touch /home/sterling/install/original/install-config.yaml
sudo chown sterling /home/sterling/install/original/agent-config.yaml
sudo chown sterling /home/sterling/install/original/install-config.yaml
sudo chmod 600 /home/sterling/install/original/agent-config.yaml
sudo chmod 600 /home/sterling/install/original/install-config.yaml
```

Copy the contents of the example agent-config.yaml and install-config.yaml to the newly created files in the /home/sterling/install/original directory. Use your preferred editor to paste content into the empty files we created. Once completed proceed with the commands below.

```bash
cp -p /home/sterling/install/original/*.* /home/sterling/install

sudo dnf install /usr/bin/nmstatectl -y

```
Make the necessary adjustments to the example files that align with your environment 
from the /home/sterling/install directory. Once all edits have been made to the agent-config.yaml & install-config.yaml we're ready to build the Agent ISO.<br />

***CAUTION***<br />
When you generate the ISO with the command below, the agent-config.yaml and the install-config.yaml are automatically deleted from the /usr/sterling/install directory as part of the agent iso creation. Please ensure that you have back up these files prior to running the command below.
```bash
openshift-install --dir=/home/sterling/install agent create image
```

## OpenShift Installation<br />

Take the Agent ISO previously created and mount it to all Control-Plane and Worker nodes via iLO, iDRAC or whatever process is used. Ensure that the boot order is set to boot from virtual cd/dvd first and power-on all servers.<br />
As the servers boot they will identify which server holds the rendezvous IP and begin registering with it.<br />

***CAUTION***<br />
Monitor the installation progress. The nodes will boot up and register themselves with the cluster API.
Once the nodes appear in the cluster as "Ready" or "NotReady" (before they are fully configured), use oc commands to apply the unique labels that match your MachineConfig selectors.

## Apply labels to match the MachineConfigs
```bash

oc label node <master-0-node-name> disk-id.config.openshift.io/node="master-0"
oc label node <master-1-node-name> disk-id.config.openshift.io/node="master-1"
oc label node <master-2-node-name> disk-id.config.openshift.io/node="master-2"
oc label node <master-2-node-name> disk-id.config.openshift.io/node="worker-0"
oc label node <master-2-node-name> disk-id.config.openshift.io/node="mworker-1"

```

## Monitor OpenShift Installtion<br />

Run the first command below below while in the /home/sterling/install directory to monitor the bootstrap process. Once the bootstrap completes you can run the second command to monitor the API availability.
```bash
openshift-install wait-for bootstrap-complete
openshift-install wait-for install-complete
```
## OpenShift Installation Complete

Once the installation is complete, navigate to the UI to ensure it's available

```
https://console-openshift-console.apps.<cluster-name>.<base-domain>/
```
The Kubeadmin password and the kubeconfig were automatically generated when the Agent ISO was created. Below we'll cat the kubeadmin_password, login into the cluster with an oc command, and export the kubeconfig.

```bash
cat /home/sterling/install/auth/kubeadmin_password
```
Copy the Kubeadmin_password displayed 

```bash
oc login --username=kubeadmin --password=your_password https://api.workcluster.yourdomain.com:6443
export KUBECONFIG=/home/sterling/install/auth/kubeconfig
```
## Podman Login

We'll use Podman to login to each of the following registries 

* docker.io
* quay.io
* registry.access.redhat.com
* registry.redhat.io

## Create Secret

```bash
mkdir /home/sterling/.config/containers/
oc create secret generic mysecret --from-file=/home/sterling/.config/containers/auth.json --type=kubernetes.io/podmanconfigjson
````
## Install Dell CSM Operator and associated PowerScale CSI
Use the following guide to properly install and test

https://dell.github.io/csm-docs/docs/getting-started/installation/openshift/powerscale/csmoperator/

## Create Certificate for API and Ingress VIP's

Below, we create a working directory and create an answer file (req.conf) for the CSR and Private Key generation

```bash

cd ~
mkdir certs
cd certs

```


##Example req.conf

```bash

[ req ]
default_bits = 2048
prompt = no
encrypt_key = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[ dn ]
CN = api.workcluster.sterling.xyz
O = YourOrganization
OU = YourOU
L = YourLocation
ST = YourState

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = api.workcluster.sterling.xyz
DNS.2 = *.apps.workcluster.sterling.xyz

```

##Create CSR and Private Key
```bash

openssl req -new -newkey rsa:4096 -nodes -keyout ocp.key -out ocp.csr -config /home/sterling/certs/req.conf

```

##Send CSR off to 3rd party for certificate generation. Once complete bring the certificate back to the machine where the CSR was generated and complete the following steps to update the cluster


```bask

oc create secret tls cluster-certs --cert=certnew.cer --key=openshift.key -n openshift-ingress

oc patch ingresscontroller default -n openshift-ingress-operator \
  --type=merge -p \
  '{"spec": {"defaultCertificate": {"name": "cluster-certs"}}}'

oc create configmap custom-ca \
  --from-file=ca-bundle.crt=/home/sterling/certs/certnew.cer \
  -n openshift-config


oc patch proxy/cluster \
  --type=merge \
  --patch='{"spec":{"trustedCA":{"name":"custom-ca"}}}'

oc create secret tls custom-api-cert --cert=/home/sterling/certs/openshift.cer --key=/home/sterling/certs/ocp.key -n openshift-config

oc patch apiserver cluster --type=merge -p='{"spec":{"servingCerts": {"namedCertificates": [{"names": ["api.workcluster.leidos.lab"], "servingCertificate": {"name":"custom-api-cert"}}]}}}'

update browser with root ca

~~~

```
###Nvidia Installation<br />
https://docs.nvidia.com/datacenter/cloud-native/openshift/25.10/install-nfd.html

For OpenShift 4.19, the correct installation order for the operators to enable high-speed networking with GPUDirect RDMA is based on their dependencies:

- Node Feature Discovery (NFD) Operator
- SR-IOV Network Operator (ensure SR-iov is enabled in bios/uefi)
- NVIDIA Network Operator
- NVIDIA GPU Operator 

Step-by-Step Installation Order<br />
Here is the recommended sequence for installing and configuring the operators:<br />

1. Install Node Feature Discovery (NFD) Operator <br />
NFD is a prerequisite for both the SR-IOV Operator and the NVIDIA GPU Operator. It automatically labels nodes with hardware capabilities (like the presence of GPUs or SR-IOV-capable NICs), which the other operators use for targeting the correct nodes.

3. Install SR-IOV Network Operator 
This operator is responsible for discovering, configuring, and provisioning the SR-IOV Virtual Functions (VFs) on your network adapters. It creates the necessary CNI plugins and device plugins for SR-IOV networking to function.

5. Install NVIDIA Network Operator 
The NVIDIA Network Operator works in conjunction with the GPU Operator to enable RDMA and GPUDirect RDMA capabilities. It handles the installation of required drivers (like Mellanox OFED) and configures the RDMA device plugin. It relies on the NFD labels to identify <br />relevant nodes and can work with the SR-IOV operator to expose RDMA-capable VFs to pods.

7. Install NVIDIA GPU Operator 
The GPU Operator manages the deployment of all necessary NVIDIA software components, including the GPU drivers, CUDA libraries, device plugins, and monitoring tools. It requires NFD to identify the GPU nodes and interacts with the Network Operator's components to<br /> enable GPUDirect RDMA functionality once all prerequisites are met. <br />

Following this order ensures that each operator's dependencies are met before its configuration begins. For detailed installation guides, refer to the official Red Hat documentation and NVIDIA documentation hub. 
