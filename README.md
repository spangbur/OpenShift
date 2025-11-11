# OpenShift Agent-based installation
# HAProxy & Keepalived
#HAProxy-01 Example Configuration
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
HAProxy-02 Example Configuration

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

# Agent-config.yaml Example
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
# Install-config.yaml Example
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
  networkType: OVNKubernetes
  machineNetwork:
    - cidr: 10.151.87.0/24
platform:
  baremetal:
    apiVIPs: 
      - 10.151.87.99
    ingressVIPs: 
      - 10.151.87.100
pullSecret: 'your secret goes here"}}}'
sshKey: |
  your ssh key goes here
```
   
