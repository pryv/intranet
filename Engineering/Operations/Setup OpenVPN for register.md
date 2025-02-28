# Setup OpenVPN link between register machines

Source: [Install OpenVPN server by Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-16-04)

This guide explains how to install a VPN server on the master register machine (the one with write access to Redis), setup a config for this.  
Then it describes how to install openVPN on the slave register machine, generating the required keys and config files.


## Install OpenVPN on master register

export DOMAIN=YOUR_PRYVIO_DOMAIN

- `apt-get update`  
- `apt-get install openvpn easy-rsa`  
- `make-cadir ~/openvpn-ca`  
- `cd ~/openvpn-ca/`  

### Setup certificate specs

- `vim vars`  

Fill these:  

```
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="SanFrancisco"
export KEY_ORG="Fort-Funston"
export KEY_EMAIL="me@myhost.mydomain"
export KEY_OU="MyOrganizationalUnit"

export KEY_NAME="server"
```

accordingly to your organisation, here is the one for pryv:  

```
export KEY_COUNTRY="CH"
export KEY_PROVINCE="Vaud"
export KEY_CITY="Morges"
export KEY_ORG="Pryv SA"
export KEY_EMAIL="admin@${DOMAIN}
export KEY_OU="IT"

export KEY_NAME="reg.${DOMAIN}"
```

- `source vars`  
- `./clean-all`  
- `./build-ca`  # confirm prompts  

- `./build-key-server reg.${DOMAIN}` # confirm prompts, ignore password fields  

- `./build-dh` # do this in another window, long process (~5-10min)
- `openvpn --genkey --secret keys/ta.key`  

### Generate key for client - start

- `export CLIENT_HOSTNAME=YOUR_CLIENT_HOSTNAME`

- `cd ~/openvpn-ca`  
- `source vars`  
- `./build-key client-${CLIENT_HOSTNAME}` # confirm prompts  

### Start with the configuration

- `cd ~/openvpn-ca/keys` 
- `cp ca.crt ca.key reg.${DOMAIN}.crt reg.${DOMAIN}.key ta.key dh2048.pem /etc/openvpn`  
- `gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz | sudo tee /etc/openvpn/server.conf`  

### Adjust server config

`vim /etc/openvpn/server.conf`  

- uncomment `tls-auth` line  
- add `key-direction 0` on the line below  
- in the ciphers, uncomment `cipher AES-128-CBC`  
- Below, add `auth SHA256`  
- uncomment `user nobody` & `group nogroup`  
- change `proto` from UDP to TCP  
- change `cert server.crt` to `reg.${DOMAIN}.crt`  
- change `key server.key` to `reg.${DOMAIN}.key`  

### Allow IP forwarding on master

- `vim /etc/sysctl.conf` # uncomment `net.ipv4.ip_forward=1`
- `sysctl -p`  

### Adjust the UFW Rules to Masquerade Client Connections

- `ip route | grep default` # read the name of the interface, found just after `dev`. We'll name it `DEFAULT_INTERFACE` afterwards  
- `vim /etc/ufw/before.rules` if the file is empty, run `apt-get install ufw` and accept to install the maintainer's version

Add the following lines below the first commented block

```
# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
# Allow traffic from OpenVPN client to ${DEFAULT_INTERFACE} (change to the interface you discovered!)
-A POSTROUTING -s 10.8.0.0/8 -o ${DEFAULT_INTERFACE} -j MASQUERADE
COMMIT
# END OPENVPN RULES
```

- `vim /etc/default/ufw` # change this `DEFAULT_FORWARD_POLICY="ACCEPT"` (from DROP to ACCEPT)  
- `ufw allow 1194/tcp`  
- `ufw allow OpenSSH`  
- `ufw disable`  
- `ufw enable` 
- `systemctl start openvpn@server`  
- `systemctl status openvpn@server` # check for line `Active: active (running) ...`  
- `ip addr show tun0` # check for VPN interface  
- `systemctl enable openvpn@server` # activate for startup at boot  

### client config infrastructure

- `mkdir -p ~/client-configs/files`  
- `chmod 700 ~/client-configs/files`  
- `cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf`  

#### specify configuration
- `vim ~/client-configs/base.conf`  

- set `remote SERVER_IP_ADDRESS 1194`  
- set `proto tcp`  
- uncomment `user nobody`  
- uncomment `group nogroup`  
- comment `ca ca.crt`  
- comment `cert client.crt`  
- comment `key client.key`  
- set `cipher AES-128-CBC`  
- set `auth SHA256`  
- set `key-direction 1`  
- add `# script-security 2`  
- add `# up /etc/openvpn/update-resolv-conf`  
- add `# down /etc/openvpn/update-resolv-conf`  

- uncomment last 3 lines if your client has the `/etc/openvpn/update-resolv-conf` file

#### create config generation script

`vim ~/client-configs/make_config.sh`  

add the following script:
```
#!/bin/bash

# First argument: Client identifier

KEY_DIR=~/openvpn-ca/keys
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn
```

- `chmod 700 ~/client-configs/make_config.sh`  
- `cd ~/client-configs`  

#### create config for client X

- `./make_config.sh client-${CLIENT_HOSTNAME}`  
- `ls ~/client-configs/files` # check for `client-${CLIENT_HOSTNAME}.ovpn` file  

- copy `client-${CLIENT_HOSTNAME}.ovpn` file to client machine

## Install openVPN on slave register

- `apt-get update`  
- `apt-get install openvpn`  

if file `/etc/openvpn/update-resolv-conf` exists on client, edit the `client-${CLIENT_HOSTNAME}.ovpn` file:
-  uncomment `script-security 2`  
- uncomment `up /etc/openvpn/update-resolv-conf`  
- uncomment `down /etc/openvpn/update-resolv-conf`

### launch

- `openvpn --config client-${CLIENT_HOSTNAME}.ovpn`

### Set it to boot on startup

- Run `vim /etc/default/openvpn` and add `AUTOSTART=client-${CLIENT_HOSTNAME}.ovpn`
- Add file `client-${CLIENT_HOSTNAME}.ovpn.conf` (**note the .conf extension**) in `/etc/openvpn/`


## Set Redis connection

### set master coordinates in slave config

In slave register, add this line to the register config file found in `${PRYV_CONF_ROOT}/reg/redis/conf/redis.conf`: `slaveof ${tun0-interface-ip-address-master} 6379`

You can find the tun0 interface IP address using `ifconfig` in master register.

### Make master redis container reachable through VPN

In the master register docker-compose `reg.yml` files, add the following to the redis service:

```
    ports:
      - "${tun0-interface-ip-address-master}:6379:6379"
```

Restart the redis containers to apply changes.

## Set healthcheck

Create a new healthcheck on [healthchecks.io](https://healthchecks.io) with a period of 5min and a 15min grace.

In slave register, run `EDITOR=vim crontab -e` and add the following lines:

```
# VPN check
*/5 * * * * ping -c 5 -q ${tun0-interface-ip-address-master} >/dev/null 2>&1 && curl -fsS --retry 3 https://hchk.io/${healthcheck-hash-code}
```


