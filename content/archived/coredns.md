+++
title = "coredns"
+++

## Server preparations

### DNSUtils
As it is a DNS Server we want some tools to debug:

```
sudo apt install dnsutils -y
```

### Service Account
For the coredns Service to run we need a system user:

```
cat <<EOF >/usr/lib/sysusers.d/coredns-sysusers.conf
u coredns - "CoreDNS is a DNS server that chains plugins" /var/lib/coredns
EOF
```

### TMP Files
And add a directory:

```
cat <<EOF >/usr/lib/tmpfiles.d/coredns-tmpfiles.conf
d /var/lib/coredns 0755 coredns coredns -
EOF
```

## Installation
Penguins are running [$CoreDNS](https://coredns.io/).

### Binary
First we need to get the binary on the system.
Head over [here](https://github.com/coredns/coredns/releases/) and copy the link for the newest `amd64` binary.

Download on system:

```
wget https://github.com/coredns/coredns/releases/download/v1.8.4/coredns_1.8.4_linux_amd64.tgz
tar -xzvf coredns_1.8.4_linux_amd64.tgz
rm coredns_1.8.4_linux_amd64.tgz
sudo mv coredns /usr/bin/
sudo chown coredns:coredns /usr/bin/coredns
sudo chmod 0755 /usr/bin/coredns
```


### Directories
All right now that the binary is there we need to create some directories for coredns to store configuration:

```
sudo mkdir -p /etc/coredns/zones
sudo chown coredns:coredns /etc/coredns
sudo chown coredns:coredns /etc/coredns/zones
sudo chmod 755 /etc/coredns
sudo chmod 755 /etc/coredns/zones
```

### systemd-service
Last thing to do is to create a systemd-unit file to run coredns:

`/etc/systemd/system/coredns.service`:
```
[Unit]
Description=CoreDNS DNS server
Documentation=https://coredns.io
After=network.target

[Service]
PermissionsStartOnly=true
LimitNOFILE=1048576
LimitNPROC=512
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
User=coredns
WorkingDirectory=~
ExecStart=/usr/bin/coredns -conf=/etc/coredns/Corefile
ExecReload=/bin/kill -SIGUSR1 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Permissions and apply:
```
sudo chmod 0644 /etc/systemd/system/coredns.service
sudo systemctl daemon-reload
sudo systemctl enable --now coredns
```

Source:  https://github.com/coredns/deployment/blob/master/systemd/coredns.service
Example: https://aur.archlinux.org/cgit/aur.git/tree/coredns.service?h=coredns-bin

## Configuration
Now that coredns is Running we need to make a config file for it so that it  keeps running.

Here's my config:

```
.:53 {

  ###### General Settings ######
  # log queries
  log
  # log errors
  errors
  # provide prometheus metrics
  prometheus .:1053
  # automatically reload the configuration from time to time
  reload

  ###### Serving DNS from file ######
  # Assume that zone files are placed relative to this path
  root /etc/coredns/zones

  ### technat.lab ###
  file db.technat.lab technat.lab

  ### core.technat.lab ###
  file db.core.technat.lab core.technat.lab
  file db.111.168.192 111.168.192.in-addr.arpa

  ### tools.technat.lab ###
  file db.tools.technat.lab tools.technat.lab
  file db.112.168.192 112.168.192.in-addr.arpa

  ### apps.technat.lab ###
  file db.apps.technat.lab apps.technat.lab
  file db.113.168.192 113.168.192.in-addr.arpa

  ### zoo.technat.lab ###
  file db.zoo.technat.lab zoo.technat.lab
  file db.114.168.192 114.168.192.in-addr.arpa

  # Forward all (.) to Hetzner Recursive DNS (v4 && v6)
  forward . 213.133.98.98 213.133.99.99 213.133.100.100 2a01:4f8:0:1::add:1010 2a01:4f8:0:1::add:9999 2a01:4f8:0:1::add:9898 {
    policy random
  }
	  ###### Primary function ######
  # allow transfer of all zones to penguin-02
  transfer . {
    to 192.168.111.12
  }
}
```

### Zone files

Because the zone files are all more or less the same I'll just list the root technat.lab zone, and one forward / reverse zone of the others.

####  technat.lab Zone
```
$TTL    3600
$ORIGIN technat.lab.
@    IN    SOA    technat.lab. technat.technat.ch. (
             1          ; Serial
             7200       ; Refresh
             3600       ; Retry
             1209600    ; Expire
             3600       ; Negative Cache TTL
)

; name servers - NS-Records
@    IN    NS    penguin-01.core.technat.lab.
@    IN    NS    penguin-02.core.technat.lab.
```

#### core.technat.lab Zone
```
$TTL    3600
$ORIGIN core.technat.lab.
@    IN    SOA    penguin-01.core.technat.lab. technat.technat.ch. (
             1         ; Serial
             7200      ; Refresh
             3600      ; Retry
             1209600   ; Expire
             3600      ; Negative Cache TTL
)

; name servers - NS records
@    IN    NS    penguin-01
@    IN    NS    penguin-02

; name servers - A records
penguin-01       IN    A    192.168.111.11
penguin-02       IN    A    192.168.111.12
coconut          IN    A    192.168.111.1
```

#### 111.168.192.in-addr.arpa
```
$TTL    3600
@    IN    SOA    penguin-01.core.technat.lab. technat.technat.ch. (
             1         ; Serial
             7200      ; Refresh
             3600      ; Retry
             1209600   ; Expire
             3600      ; Negative Cache TTL
)

; name servers - NS records
@    IN    NS    penguin-01
@    IN    NS    penguin-02

; PTR records
11   IN    PTR   penguin-01
12   IN    PTR   penguin-02
1    IN    PTR   coconut
```

#### tools.technat.lab zone
```
$TTL    3600
$ORIGIN tools.technat.lab.
@    IN    SOA    penguin-01.core.technat.lab. technat.technat.ch. (
                  1      ; Serial
             604800      ; Refresh
              86400      ; Retry
            2419200      ; Expire
             604800 )    ; Negative Cache TTL
;

; name servers - NS records
@    IN    NS    penguin-01.core.technat.lab.
@    IN    NS    penguin-02.core.technat.lab.

; name servers - A records
coconut          IN    A    192.168.112.1
```

### Secondary
The penguin-02 is also configured with the systemd-service and coredns user but has a different Corefile:

```
.:53 {

  ###### General Settings ######
  # log queries
  log
  # log errors
  errors
  # provide prometheus metrics
  prometheus .:1053
  # automatically reload the configuration from time to time
  reload

  ###### Serving DNS from master ######
  secondary technat.lab core.technat.lab tools.technat.lab apps.technat.lab zoo.technat.lab 111.168.192.in-addr.arpa 112.168.192.in-addr.arpa 113.168.192.in-addr.arpa 114.168.192.in-addr.arpa {
    transfer from 192.168.111.11
  }

  # Forward all (.) to Hetzner Recursive DNS (v4 && v6)
  forward . 213.133.98.98 213.133.99.99 213.133.100.100 2a01:4f8:0:1::add:1010 2a01:4f8:0:1::add:9999 2a01:4f8:0:1::add:9898 {
    policy random
  }
}
```

Zones are transfered from primary so no zones have to be defined.

### Further reading
* [very good blog article[(https://blog.idempotent.ca/2018/04/18/run-your-own-home-dns-on-coredns/)
*

++++++--
## old
penguin-01 and penguin-02 are DNS Servers on Debian LXC images. they are working in a primary / secondary mode where penguin-01 is  the primary.

## Installation

BIND:
```
sudo apt install bind9 bind9utils bind9-doc -y
```

## BIND-Config
For both DNS servers there are two config files that have to be configured.

### penguin-01
Configured as primary DNS Server

#### named.conf.options
Basically just replace the entire file (`/etc/bind/named.conf.options`) with the following:
```
options {

        // +++++++++ General Settings +++++++++++++++

				// DNS-Cache directory
        directory "/var/cache/bind";

				 // In general we want dnssec validation, but we don't care about details
				 // More informations: https://kb.isc.org/docs/aa-01182
        dnssec-validation auto;

        // Open your ears and hear for traffic!
        listen-on port 53 {
                192.168.111.11;
								127.0.0.1;
        };

				// disable zone-transfer per default (security feature)
				allow-transfer {
				  none;
		    };

				// +++++++++- Primary Settings +++++++++++++++

        // ++++++-- DNS Resolver (Recursive Requests) +++++++++--
        //Allow Queries in general
        allow-query {
                technat.lab;
        };
        //Allow Queries to cache
        allow-query-cache {
                technat.lab;
        };
        //Allow recursive queries
        allow-recursion {
                technat.lab;
        };
				// Forward unknown things to Hetzner DNS
        forwarders {
                213.133.98.98;
                213.133.99.99;
                213.133.100.100;
        };


};

// Subnets of technat.lab defined, just to prevent malicous requests
acl "technat.lab" {
        192.168.111.0/24;
        192.168.112.0/24;
        192.168.113.0/24;
        192.168.62.0/24;
				192.168.63.0/24;
        localhost;
        localnets;
};
```

#### named.conf.local
Basically just replace the entire file (`/etc/bind/named.conf.local`) with the following:

```
zone "technat.lab" {
        type master;
				file "/etc/bind/zones/db.lab.technat";
        allow-transfer { 192.168.111.12; };
};

zone "111.168.192.in-addr.arpa" {
				type master;
				file "/etc/bind/zones/db.111.168.192.in-addr.arpa";
				allow-transfer { 192.168.111.12; };
};

	zone "112.168.192.in-addr.arpa" {
				type master;
				file "/etc/bind/zones/db.112.168.192.in-addr.arpa";
				allow-transfer { 192.168.111.12; };
	};

	zone "113.168.192.in-addr.arpa" {
				type master;
				file "/etc/bind/zones/db.113.168.192.in-addr.arpa";
				allow-transfer { 192.168.111.12; };
	};
```

#### db.lab.technat
Basically just replace the entire file (`/etc/bind/zones/db.lab.technat`) with the following:
```
;
; BIND data file for technat.lab
;
$TTL    604800
@       IN      SOA     technat.lab. root.technat.lab. (
                              3         ; Serial
                             604800         ; Refresh
                             86400         ; Retry
                             2419200         ; Expire
                             604800 )       ; Negative Cache TTL
;
; name servers - NS records
IN NS penguin-01.technat.lab.
IN NS penguin-02.technat.lab.

; name servers A records
penguin-01.technat.lab. IN A 192.168.111.11
penguin-02.technat.lab. IN A 192.168.111.12

; technat.lab A records CoreInfrastructure
coconut         IN      A       192.168.111.1


jellyfish       IN      A       192.168.112.3
git             IN      A       192.168.112.3
kuriosigans     IN      A       192.168.112.253
kuriosiente     IN      A       192.168.112.254

leopard-01      IN      A       192.168.111.6
registry        IN      A       192.168.112.3
wolt            IN      A       192.168.112.100

; technat.lab A records Infrastructure


; technat.lab A records zoo.technat.lab

; technat.lab A records children-zoo.technat.lab

```

#### db.111.168.192.in-addr.arpa
```
$TTL    604800
@       IN      SOA     technat.lab. root.technat.lab. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; name servers
      IN      NS      penguin-01.technat.lab.
      IN      NS      penguin-02.technat.lab.

; name servers PTR records
11 IN PTR penguin-01.technat.lab.
12 IN PTR penguin-02.technat.lab.


; technat.lab PTR records CoreInfrastructure
1 IN PTR coconut


3 IN PTR jellyfish

6 IN PTR leopard-01

; technat.lab PTR records Infrastructure
```

#### db.112.168.192.in-addr.arpa
```
$TTL    604800
@       IN      SOA     technat.lab. root.technat.lab. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; name servers
      IN      NS      penguin-01.technat.lab.
      IN      NS      penguin-02.technat.lab.

; technat.lab PTR records CoreInfrastructure
1 IN PTR coconut

; technat.lab PTR records Infrastructure

3 IN PTR jellyfish
3 IN PTR git
253 IN PTR kuriosigans
254 IN PTR kuriosiente

3 IN PTR registry
100 IN PTR wolt
```

#### db.113.168.192.in-addr.arpa
```
$TTL    604800
@       IN      SOA     technat.lab. root.technat.lab. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; name servers
      IN      NS      penguin-01.technat.lab.
      IN      NS      penguin-02.technat.lab.

; technat.lab PTR records CoreInfrastructure
1 IN PTR coconut

; technat.lab PTR records Infrastructure

; technat.lab PTR records zoo.technat.lab

; technat.lab PTR records children-zoo.technat.lab
```

### penguin-02
Configured as secondary DNS server

#### named.conf.options
Basically just replace the entire file (`/etc/bind/named.conf.options`) with the following:
```
options {

        // +++++++++ General Settings +++++++++++++++

				// DNS-Cache directory
        directory "/var/cache/bind";

				 // In general we want dnssec validation, but we don't care about details
				 // More informations: https://kb.isc.org/docs/aa-01182
        dnssec-validation auto;

        // Open your ears and hear for traffic!
        listen-on port 53 {
                any;
        };

				// +++++++++- Secondary Settings +++++++++++++++

        // ++++++-- DNS Resolver (Recursive Requests) +++++++++--
        //Allow Queries in general
        allow-query {
                technat.lab;
        };
        //Allow Queries to cache
        allow-query-cache {
                technat.lab;
        };
        //Allow recursive queries
        allow-recursion {
                technat.lab;
        };
				// Forward unknown things to Hetzner DNS
        forwarders {
                213.133.98.98;
                213.133.99.99;
                213.133.100.100;
        };


};

// Subnets of technat.lab defined, just to prevent malicous requests
acl "technat.lab" {
        192.168.111.0/24;
        192.168.112.0/24;
        192.168.113.0/24;
        192.168.62.0/24;
				192.168.63.0/24;
        localhost;
        localnets;
};
```

#### named.conf.local
Basically just replace the entire file (`/etc/bind/named.conf.local`) with the following:

```
zone "technat.lab" {
        type slave;
				file "db.lab.technat";
				masters { 192.168.111.11; };
};

zone "111.168.192.in-addr.arpa" {
				type slave;
				file "db.111.168.192.in-addr.arpa";
				masters { 192.168.111.11; };
};

	zone "112.168.192.in-addr.arpa" {
				type slave;
				file "db.112.168.192.in-addr.arpa";
				masters { 192.168.111.11; };
	};

	zone "113.168.192.in-addr.arpa" {
				type slave;
				file "db.113.168.192.in-addr.arpa";
				masters { 192.168.111.11; };
	};
```

## Sources
* https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-18-04
