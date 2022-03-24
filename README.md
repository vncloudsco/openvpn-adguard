<h3 align="center">
    <img src="https://camo.githubusercontent.com/04c8294cd6a8e01ebd5fba2a65747ab574f7c2e75df942e53fa76d16f4c38005/68747470733a2f2f63646e2e616467756172642e636f6d2f7075626c69632f416467756172642f436f6d6d6f6e2f616467756172645f686f6d652e737667" alt="Logo" width="500">
</h3>

<h3 align="center">
    <a href="https://openvpn.net">OpenVPN</a> and <a href="https://github.com/AdguardTeam/AdGuardHome">AdGuardHome</a> wrapped up in a docker-compose setup
</h3>

<br/>

<br>

This project is a composition out of the official [AdGuardHome docker image](https://github.com/AdguardTeam/AdGuardHome) and a hand-crafted [openvpn-image](openvpn-docker) to set up a ready-to-use
VPN with AdGuardHome as dns-resolve in less than a minute. Its configuration is kept simple, you can add / remove clients and easily extend it as the configuration is stored in a centralized and easily manageable way.
Enjoy!

### Setup

First clone this repository:

```sh
git clone https://github.com/vncloudsco/openvpn-adguard.git
cd openvpn-adguard
```

Make sure you're using the latest [docker](https://docs.docker.com/get-docker/) and [docker-compose](https://docs.docker.com/compose/install/).
I'm using `v3.5` for the [docker-compose.yml](docker-compose.yml) so you'll need at least `v17.12.0` for the docker-ngine (see [this table](https://docs.docker.com/compose/compose-file/#compose-and-docker-compatibility-matrix)).


After you've installed all the pre-requisites you can run.
```sh
sudo docker-compose up -d
```

After this is done you'll find two new folders inside of this repository - the `/openvpn` folder will contain all of your certificates as well as an easy-rsa configuration file.

> Until [this issue](https://github.com/moby/moby/issues/32582) has been resolved I'll be using mounted host directories for the sake of simplicity.

> If you're using a VPS make sure to open 1194/udp!

#### Generating `.ovpn` files

> Before you generate any client certificate you must update the host in [client configuration](openvpn/config/client.conf).
> This file will be used as base-configuration for each `.ovpn` file! You probably at least want to change the IP address to your public one.

```sh
sudo docker exec openvpn bash /opt/app/bin/genclient.sh <name> <password?>
```

You can find you `.ovpn` file under `/openvpn/clients/<name>.ovpn`, make sure to change the remote ip-address / port / protocol.

#### Revoking `.ovpn` files

```sh
sudo docker exec openvpn bash /opt/app/bin/rmclient.sh <name>
```

Revoked certificates won't kill active connections, you'll have to restart the service if you want the user to immediately disconnect:
```sh
sudo docker-compose restart openvpn
```


### Configuration

#### OpenVPN
Configuration files (such as [`server.conf`](openvpn/config/server.conf) and [`client.conf`](openvpn/config/client.conf)) are stored in [openvpn/config](openvpn/config).
They get copied every time the instance gets spawned so feel free to change / update them any time.

#### AdGuardHome
We're always using the very latest AdGuardHome version - start the AdGuardHome service at least once to edit configuration files manually.


### FAQ & Recipes

#### Launching multiple openvpn instances with different protocol/port config
First copy the [openvpn](openvpn) directory including [openvpn/config](openvpn/config) (copy just the `config` folder!), then add another service to [docker-compose.yml](docker-compose.yml).

Example assuming we want to name our second openvpn instance `openvpn-tcp-443`:
```sh
mkdir openvpn-tcp-443
cp -r openvpn/config openvpn-tcp-443
```

You can now make changes to our new config files in `openvpn-tcp-443/config`. Change `proto` to `tcp` and `port` to `443`,
you'll also need to comment out `explicit-exit-notify 1` as this is only compatible with `proto udp` (update both `server.conf` and `client.conf`!).

Now add our new service:
```yaml
# ... other services
    openvpn-tcp-443:
        container_name: openvpn-tcp-443
        build: ./openvpn-docker
        ports:
            - 443:443/tcp
        volumes:
            - ./openvpn/pki:/etc/openvpn/pki # Keep the PKI
            - ./openvpn-tcp-443/clients:/etc/openvpn/clients
            - ./openvpn-tcp-443/config:/etc/openvpn/config # !! We're using our second configuraion
        cap_add:
            - NET_ADMIN
        restart: unless-stopped
# ... other services
```

> Keep in mind that if you want to generate a client-config for that service  we've just made you'll
> have to use the openvpn-tcp-443 container e.g. `sudo docker exec openvpn-tcp-443 bash /opt/app/bin/genclient.sh <name>`.

### Troubleshooting

#### Port 53 is already in use

> ERROR: for AdGuardHome  Cannot start service AdGuardHome: driver failed programming external connectivity on endpoint AdGuardHome (...): Error starting userland proxy: listen tcp 0.0.0.0:53: bind: address already in use

You'll need to disable the local dns-server, see [this](https://askubuntu.com/questions/907246/how-to-disable-systemd-resolved-in-ubuntu) and [this](https://askubuntu.com/questions/191226/dnsmasq-failed-to-create-listening-socket-for-port-53-address-already-in-use) askubuntu thread.
You can stop, disable and mask the `systemd-resolved` service using the following commands:
```sh
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo systemctl mask systemd-resolved
```