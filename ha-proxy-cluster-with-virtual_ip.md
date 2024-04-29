# HA Proxy for K8s Cluster
## Node Environment
|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Load Balancer|loadbalancer1.example.com|172.16.16.51|Ubuntu 20.04|512M|1|
|Load Balancer|loadbalancer2.example.com|172.16.16.52|Ubuntu 20.04|512M|1|

##### Noted: virtual_ipaddress is : 172.16.16.100

## Set up load balancer nodes (loadbalancer1 & loadbalancer2)
##### Connect to loadbalancer1 & loadbalancer2
```
ssh root@172.16.16.51
ssh root@172.16.16.52
```
##### Install Keepalived & Haproxy
```
apt update && apt install -y keepalived haproxy

```

##### Configure keepalived/On both nodes create the health check script /etc/keepalived/check_apiserver.sh
```
cat >> /etc/keepalived/check_apiserver.sh <<EOF
#!/bin/sh

errorExit() {
  echo "*** $@" 1>&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 172.16.16.100; then
  curl --silent --max-time 2 --insecure https://172.16.16.100:6443/ -o /dev/null || errorExit "Error GET https://172.16.16.100:6443/"
fi
EOF

chmod +x /etc/keepalived/check_apiserver.sh

```
##### Create keepalived config /etc/keepalived/keepalived.conf
```
cat >> /etc/keepalived/keepalived.conf <<EOF
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        172.16.16.100
    }
    track_script {
        check_apiserver
    }
}
EOF

```
##### Enable & start keepalived service
```
systemctl enable --now keepalived

```

##### Check log status serive
```
journalctl -flu keepalived

```
##### Check IP address
```
ip a s
ip a s eth1

```

==============================================================================
## Configure haproxy
##### Update **/etc/haproxy/haproxy.cfg**
```
cat >> /etc/haproxy/haproxy.cfg <<EOF

frontend kubernetes-frontend
  bind *:6443
  mode tcp
  option tcplog
  default_backend kubernetes-backend

backend kubernetes-backend
  option httpchk GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  balance roundrobin
    server kmaster1 172.16.16.101:6443 check fall 3 rise 2
    server kmaster2 172.16.16.102:6443 check fall 3 rise 2
    server kmaster3 172.16.16.103:6443 check fall 3 rise 2

EOF

```
##### Enable & restart haproxy service
```
systemctl enable haproxy && systemctl restart haproxy

```
==============================================================================