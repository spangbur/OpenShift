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

```
mkdir install
mkdir /install/original

touch /install/original/agent-config.yaml
touch /install/original/install-config.yaml
```

Copy the contents of the example agent-config.yaml and install-config.yaml to the newly created files in the /install/original directory

```
cp /install/original/*.* /install
```
Make the necessary adjustments to the example files that align with the environment 

from the /install directory execute the following command to generate the agent iso
```
open-shift-install --dir=/install agent create image
```
