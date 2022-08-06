# Openvpn Server with Docker

In this document, and for running openvpn server with docker we use [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn) repository. 

For more info about how to use this docker compose, look at this [documentation](https://github.com/kylemanna/docker-openvpn/blob/main/docs/docker-compose.md)

This document is based on [this medium post](https://medium.com/@gurayy/set-up-a-vpn-server-with-docker-in-5-minutes-a66184882c45).

### Initializing Configuration Files

- Clone the   [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn) repository and cd into its folder

  ```sh
  git clone https://github.com/kylemanna/docker-openvpn.git
  cd docker-openvpn
  ```

- Build docker image (here we named the image `openvpn`)

  ```sh
  docker build -t openvpn .
  ```

- Create a folder which we will use to store our configuration files and keys. we need to create a file named `vars` in it

  ```sh
  mkdir ~/vpn-data && touch ~/vpn-data/vars
  ```

- Now using the `openvpn` image we built earlier, use the following command to generate Openvpn config file. Note that we mounted `vpn-data` folder we created in previous step. if your folder is in another address, mount accordingly. 

  Please look at [DNS problem section](#dns-problem) before if your VPS is on Swissmade, or if your VPScan not access public DNS servers (e.g. `8.8.8.8`)

  ```sh
  $ docker run -v ~/vpn-data:/etc/openvpn --rm openvpn ovpn_genconfig -u udp://IP_ADDRESS:3333
  Processing PUSH Config: 'block-outside-dns'
  Processing Route Config: '192.168.254.0/24'
  Processing PUSH Config: 'dhcp-option DNS 8.8.8.8'
  Processing PUSH Config: 'dhcp-option DNS 8.8.4.4'
  Successfully generated config
  Cleaning up before Exit ...
  ```

  Change the IP_ADDRESS. Also you can use any port you want. Here we used `3333`.

- We should initialize our PKI. This covers generating our CA certificate and we will have a private key belong to the PKI. We will be asked a password for protecting the private key. generate a secure password and store it in the password the passwords Excel file.

  ```sh
  docker run -v ~/vpn-data:/etc/openvpn --rm -it openvpn ovpn_initpki
  ```

- Finally, we can run the VPN server based on that config using `docker-compose.yml` (change the port, volumes and image accordingly).

  ```
  version: '2.1'
  services:
    openvpn:
      sysctls:
       - net.ipv4.ip_forward=1
      cap_add:
       - NET_ADMIN
      image: openvpn
      container_name: openvpn
      ports:
       - "3333:1194/udp"
      restart: unless-stopped
      logging:
        driver: "json-file"
        options:
          max-size: "100m"
      volumes:
       - /home/whatever/vpn-data:/etc/openvpn
  ```

  ```sh
  docker-compose up -d
  ```

  Now we can create and add users

### Adding User clients

- Add a client with following command. `user1` is the username of the client you create:

  ```sh
  docker run -v ~/vpn-data:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full user-test
  ```

  You will be prompted to enter a password for user. You can avoid setting password for the user with passing `nopass` option in the command. but it is not advised, anyone with that config file can connect to your server.

  Also, you need to enter the password for your CA password (that you set while initializing PKI). If you don't have access to the CA password, look for it in the password Excel file.

- At the last step, we will generate a configuration file which will be sent to the user.

  ```sh
  docker run -v ~/vpn-data:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient user-test > user-test.ovpn
  ```

### Revoking User Access

Revoke access of user with following command:

```sh
# Keep the corresponding crt, key and req files.
docker-compose run --rm openvpn ovpn_revokeclient $CLIENTNAME
# Remove the corresponding crt, key and req files.
docker-compose run --rm openvpn ovpn_revokeclient $CLIENTNAME remove
```



### DNS Problem

Some VPS providers block DNS server traffics for DDoS Protection purposes, like swissmade. If this is the case, your OpenVpn server can not access public DNS servers and you should provide it with the DNS server addresses that can be accessed from your VPS. Get those DNS servers from VPS support. for swissmade DNS servers are

```
46.28.201.21
46.28.201.22
```

You can provide DNS servers when generating Openvpn config files with `ovpn_genconfig` command using `-n` option. But in our first attempt to create openvpn servers we did not know this problem exists, and created openvpn without these DNS servers and it tries public DNS addresses ( e.g. `8.8.4.4`).



