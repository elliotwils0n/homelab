# HA setup
For high availability configure keepalived with haproxy.
Keepalived is used for setting up multiple load balancers with shared VIP.
HAProxy is used for load balancing k8s api servers and http(s) traffic.
For multiple load balancers, make VI_HA state SLAVE and lower its priority.

## keepalived
`/etc/keepalived/keepalived.conf`:
```
vrrp_instance VI_HA {
    state MASTER
    interface enp0s3
    virtual_router_id 210
    priority 200
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 210210
    }
    virtual_ipaddress {
        192.168.0.210/24
    }
}
```

## haproxy
append to `/etc/haproxy/haproxy.cfg`:
```
frontend k8s_api_frontend
    bind 192.168.0.210:6443
    mode tcp
    option tcplog
    default_backend k8s_api_backend
backend k8s_api_backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 192.168.0.211:6443 check
    #server master2 192.168.0.212:6443 check

frontend http_frontend
    bind 192.168.0.210:80
    mode tcp
    option tcplog
    default_backend http_backend
backend http_backend
    mode tcp
    balance roundrobin
    option tcp-check
    server worker1 192.168.0.211:30080 check
    #server worker2 192.168.0.212:30080 check

frontend https_frontend
    bind 192.168.0.210:443
    mode tcp
    option tcplog
    default_backend https_backend
backend https_backend
    mode tcp
    balance roundrobin
    option tcp-check
    server worker1 192.168.0.211:30443 check
    #server worker2 192.168.0.212:30443 check
```
