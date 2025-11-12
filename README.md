# OpenShift Agent-based installation
The Installation below is based off of OpenShift 4.19.7. An Agent-based installation was chosen to quickly intergrate NVIDIA GPU's and external Dell PowerScale storage for PV creation

## Ubuntu Configuration

Ubuntu version 24.04.3

Install Ubuntu on two servers. Configure with IP's from the OpenShift Machine Network subnet. 

After installation run the following commands:
```
sudo apt update -y && sudo upgrade -y
sudo apt install haproxy keepalived -y

sudo apt update -y
```
## HAProxy & Keepalived

### HAProxy-01 Example Configuration

```
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
sterling@haproxy-01:/etc/haproxy$
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
   default_backend controlplaneapi

frontend apiinternal
   bind *:22623
   default_backend controlplaneapiinternal

frontend secure
   bind *:443
   default_backend secure

frontend insecure
   bind *:80
   default_backend insecure

#---------------------------------------------------------------------
# static backend
#---------------------------------------------------------------------

backend controlplaneapi
   balance source
   server controller-01 10.151.87.20:6443 check
   server controller-02 10.151.87.21:6443 check
   server controller-03 10.151.87.22:6443 check


backend controlplaneapiinternal
   balance source
   server controller-01 10.151.87.20:22623 check
   server controller-02 10.151.87.21:22623 check
   server controller-03 10.151.87.22:22623 check

backend secure
   balance source
   server work-01 10.151.87.23:443 check
   server work-02 10.151.87.24:443 check

backend insecure
   balance source
   server work-01 10.151.87.23:80 check
   server work-02 10.151.87.24:80 check
```

### HAProxy-01 Keepalived Example Configuration

```
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
        auth_pass YOUR PASSWORD HERE    # Must match on all load balancers
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

### HAProxy-02 Example Configuration

```
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   https://www.haproxy.org/download/1.8/doc/configuration.txt--------
# static backend
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
globalver controller-02 10.151.87.21:6443 check
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog.87.20:22623 check
    #rver controller-02 10.151.87.21:22623 check
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #lance source
    #    local2.*                       /var/log/haproxy.log
    #rver work-02 10.151.87.24:443 check
    log         127.0.0.1 local2
backend insecure
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pideck
    maxconn     4000.151.87.24:80 check
    user        haproxyc/haproxy$
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
    default_backend controlplaneapi

frontend apiinternal
    bind *:22623
    default_backend controlplaneapiinternal

frontend secure
    bind *:443
    default_backend secure

frontend insecure
    bind *:80
    default_backend insecure

#---------------------------------------------------------------------
# static backend
#---------------------------------------------------------------------

backend controlplaneapi
    balance source
    server controller-01 10.151.87.20:6443 check
    server controller-02 10.151.87.21:6443 check
    server controller-03 10.151.87.22:6443 check


backend controlplaneapiinternal
    balance source
    server controller-01 10.151.87.20:22623 check
    server controller-02 10.151.87.21:22623 check
    server controller-03 10.151.87.22:22623 check

backend secure
    balance source
    server work-01 10.151.87.23:443 check
    server work-02 10.151.87.24:443 check

backend insecure
    balance source
    server work-01 10.151.87.23:80 check
    server work-02 10.151.87.24:80 check
```
### HAProxy-02 Keepalivd Example Configuration

```
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
        auth_pass YOUR PASSWORD HERE    # Must match on all load balancers
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
## Agent-config.yaml Example
```
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
          ipv4:
            enabled: true
            address:
              - ip: 10.151.87.20
                prefix-length: 24
      dns-resolver:
        config:
          server:
            - 10.151.87.250
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
          ipv4:
            enabled: true
            address:
              - ip: 10.151.87.21
                prefix-length: 24
      dns-resolver:
        config:
          server:
            - 10.151.87.250
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
          ipv4:
            enabled: true
            address:
              - ip: 10.151.87.22
                prefix-length: 24
      dns-resolver:
        config:
          server:
            - 10.151.87.250
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
          ipv4:
            enabled: true
            address:
              - ip: 10.151.87.23
                prefix-length: 24
      dns-resolver:
        config:
          server:
            - 10.151.87.250
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
          ipv4:
            enabled: true
            address:
              - ip: 10.151.87.24
                prefix-length: 24
      dns-resolver:
        config:
          server:
            - 10.151.87.250
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.151.87.1
            next-hop-interface: ens32
```
## Install-config.yaml Example
```
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
  serviceNetwork:
  - 172.30.0.0/16
  networkType: OVNKubernetes
platform:
  baremetal:
    apiVIPs: 
      - 10.151.87.99
    ingressVIPs: 
      - 10.151.87.100
pullSecret: 'your pull secret here'
sshKey: 'your ssh public key here'
```

## Create Agent ISO

The Agent ISO can be created on Windows, MacOS, or Linux. I've chosen RHEL Developer version which is free for illustration purposes. I chose RHEL because getting NMState installed is more straight forward than using other distro's. Use the following URL to identify and download the specific version of OpenShift tools required for your installation:

https://mirror.openshift.com/pub/openshift-v4/clients/ocp/

Requirements:<br />
openshift-client-linux-4.19.17.tar.gz<br />
openshift-install-linux-4.19.17.tar.gz

```
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
chmod 777 /home/sterling/install/original/agent-config.yaml install-config.yaml

```

Copy the contents of the example agent-config.yaml and install-config.yaml to the newly created files in the /home/sterling/install/original directory

```
cp /home/sterling/install/original/*.* /home/sterling/install

sudo dnf install nmstate -y
sudo systemctl stop firewalld
```
Make the necessary adjustments to the example files that align with the environment 

from the /home/sterling/install directory execute the following command to generate the agent iso<br />

***CAUTION***<br />
When you generate the ISO with the command below, the agent-config.yaml and the install-config.yaml are automatically deleted from the /usr/sterling/install directory as part of the agent iso creation. Please ensure that you have back up these files prior to running the command below.
```
openshift-install --dir=/home/sterling/install agent create image
```

##OpenShift Installation<br />

Take the Agent ISO previously created and mount it to all Control-Plane and Worker nodes via iLO, iDRAC or whatever process is used. Ensure that the boot order is set to boot from virtual cd/dvd first and power-on all servers.<br />
As the servers boot they will identify which server holds the rendezvous IP and begin registering with it.<br />

## Monitor OpenShift Installtion<br />

Run the command below while in the /home/sterling/install directory to monitor the bootstrap process. It is expected to see connection refused and timeout errors during the initial installation. You have to provide time for the OS and OpenShift to be installed on all machines
```
openshift-install wait-for bootstrap-complete --dir=/home/sterling/install --log-level debug
```


Monitor the installtion from the machine which was used to create the Agent ISO. Login or ssh into the machine and navigate to the /usr/sterling/install directory. You can monitor diffrent point of the installation process or installation as a whole.<br />
The openshift-install wait-for command is a crucial part of the OpenShift Container Platform installation process, particularly when using the openshift-install utility. Its primary function is to monitor and wait for specific stages of the OpenShift cluster<br /> installation to reach a complete and stable state.<br />

***Here's how it works:***<br />

*Installation Stages: The OpenShift installation process involves multiple distinct stages, such as bootstrapping the cluster, bringing up control plane nodes, deploying core components, and finally, making the cluster fully operational.<br />
*wait-for Subcommands: The openshift-install wait-for command is typically used with a subcommand that specifies which stage to wait for. The most common one is openshift-install wait-for install-complete.<br />

*Monitoring Cluster State: When executed, the wait-for command continuously monitors the state of the OpenShift cluster by interacting with its components and APIs. It checks for the successful completion of the specified installation stage.<br />

*Timeouts and Retries: The command usually has a built-in timeout period. If the specified stage does not complete within this timeframe, the command will exit, potentially indicating an issue with the installation. However, it can often be re-run if a timeout occurs,<<br /> 
allowing the installation to continue if the underlying issue was temporary or has been resolved. Signaling Completion: Once the monitored installation stage is successfully completed and the cluster reaches the desired stable state, the openshift-install wait-for<br /> command exits with a success status, signaling that the installation has progressed to that point.

```

```
