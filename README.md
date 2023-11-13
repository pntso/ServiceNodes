# Setting up a Pantos Service Node


## Prerequisites 
#### Minimum hardware requirements for a Pantos Service Node instance:
>CPU: At least a single-core CPU

>Memory (RAM): A minimum of 2 GB of RAM

>Storage: At least 8 GB of available storage space

Please note that these are the minimum hardware requirements, and actual hardware needs may vary depending on the specific workload and usage patterns of the service node. It's recommended to provide additional resources if the software will be handling a constant high volume of data.
To install and run your own Pantos Service Node instance, you need a server (e.g. a virtual private server hosted by an Internet service provider) running a current GNU/Linux distribution. 

The easiest way to get started and set up your Service Node is by using a Debian or Ubuntu server. We recommend either Debian Stable (currently Debian 12) or Ubuntu LTS (currently Ubuntu 22.04 LTS). More experienced users can also use other GNU/Linux distributions to set up their Service Node instance.

For this guide, we assume that you are familiar with GNU/Linux in general (including the most important shell commands) and you have (at least) basic system administration and networking skills. In the first section of this guide, we cover the recommended installation of a Pantos Service Node instance using the package manager of Debian/Ubuntu. In the second section, we describe how to install your Service Node instance manually on Debian/Ubuntu. Advanced users may adapt these steps to set up a Service Node instance on other GNU/Linux distributions (or even other operating systems).

## Step 1 - Installation Steps
Downloading and Installing the Service Node

**Access your server and download the Pantos Service Node package:**

`File will be shared via Telegram, please do not upload it anywhere and only transfer it via an encrypted connection e.g SFTP/SCP`

**Install the Service Node package using the apt package manager (substitute VERSION and REVISION with the actual numbers):**

`sudo apt install ./pantos-service-node-testnet_VERSION-REVISION_all.deb`

This will install the Service Node and its dependencies, including Apache, PostgreSQL, and RabbitMQ. 

It will also set up the required users, directories, access rights, databases, and message broker host. The Service Node is not started automatically since you have to perform a few configuration steps manually first.

## Step 2: Generating an SSL Certificate for the Domain
### Install Certbot

**Install Certbot (the Let's Encrypt client) on your server:**

`sudo apt update
 sudo apt install certbot`

**Obtain a Certificate**

Obtain your SSL certificate by running Certbot with the standalone plugin, which will temporarily start a web server on port 80 to perform the domain verification:
> Stop the Apache server if you are having issues executing this step

`sudo systemctl stop apache2`

`sudo certbot certonly --standalone -d yourdomain.com`


Replace yourdomain.com with your actual domain name. You'll need to ensure that port 80 on your server is open and that your DNS settings are correctly configured to point to your server's IP address.

**Certificate Location**

The SSL certificate and private key will be stored in the following directory:

`/etc/letsencrypt/live/yourdomain.com/`

Here, you'll find fullchain.pem (the certificate plus the chain) and privkey.pem (the private key).

Copy these two files to the appropriate directory, so that the Pantos Service Node can access them:

```
# Don't forget to adjust your domain name

sudo cp /etc/letsencrypt/live/yourdomain.com/fullchain.pem /etc/pantos-service-node-fullchain.pem
sudo cp /etc/letsencrypt/live/yourdomain.com/privkey.pem /etc/pantos-service-node-privkey.pem
sudo chown pantos:pantos /etc/pantos-service-node-fullchain.pem
sudo chown pantos:pantos /etc/pantos-service-node-privkey.pem
sudo chmod 400 /etc/pantos-service-node-fullchain.pem
sudo chmod 400 /etc/pantos-service-node-privkey.pem
```

### Automating Certificate Renewal and Deployment

**Renewal Job**

Certbot will automatically set up a cron job or systemd timer to attempt to renew certificates that are near expiration. You can test the renewal process with:

`sudo certbot renew --dry-run`

**Post-Renewal Hook Script**

Create a script that will handle the deployment of the new certificate after renewal. Create a file with the following script:

`sudo nano /usr/local/bin/deploy_cert.sh`

**Insert the following code into deploy_cert.sh:**

```#!/bin/bash

DOMAIN="yourdomain.com"
PANTOS_CERT_PATH="/etc/pantos-service-node-fullchain.pem"
PANTOS_KEY_PATH="/etc/pantos-service-node-privkey.pem"

PANTOS_CERT_DIR="/etc"

# Stop the Pantos service before updating certificates
sudo systemctl stop pantos-service-node-server
sudo systemctl stop pantos-service-node-celery

# Copy the new cert and private key
cp "/etc/letsencrypt/live/$DOMAIN/fullchain.pem" "$PANTOS_CERT_PATH"
cp "/etc/letsencrypt/live/$DOMAIN/privkey.pem" "$PANTOS_KEY_PATH"

# Set permissions
sudo chown pantos:pantos /etc/pantos-service-node-fullchain.pem
sudo chown pantos:pantos /etc/pantos-service-node-privkey.pem
sudo chmod 400 /etc/pantos-service-node-fullchain.pem
sudo chmod 400 /etc/pantos-service-node-privkey.pem

# Start the Pantos service again
sudo systemctl start pantos-service-node-server
sudo systemctl start pantos-service-node-celery

```

Replace yourdomain.com with your actual domain and adjust the paths if needed.

**Make the script executable:**

`sudo chmod +x /usr/local/bin/deploy_cert.sh`

**Certbot Hook Configuration**

Instruct Certbot to use this script after renewal by editing the renewal configuration file located at `/etc/letsencrypt/renewal/yourdomain.com.conf`. 

**Add a post_hook directive under the [renewalparams] section:**

`[renewalparams]
...
post_hook = /usr/local/bin/deploy_cert.sh`

Save and close the file. Certbot will now execute deploy_cert.sh after each successful renewal.

## Step 3: Generating a Key with Geth for Ethereum-compatible Wallet
### Installing Geth

**Install Geth on your server:**
```
sudo apt-get install software-properties-common
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt update
sudo apt install ethereum
```

**Generate Ethereum Wallet**
    
Generate a new Ethereum account (this will create a new keypair):

`geth account new`

You will be prompted to create a passphrase. Keep it safe and remember it, as it is needed for the service node configuration.

**Secure the Keystore File**

Locate the keystore file created by geth, which is typically located in the directory **~/.ethereum/keystore.** Move this file to the location expected by the Pantos service node:
```
sudo mv ~/.ethereum/keystore/UTC--<timestamp>--<your-address> /etc/pantos-service-node.keystore
sudo chown pantos:pantos /etc/pantos-service-node.keystore
sudo chmod 400 /etc/pantos-service-node.keystore
```

Replace **timestamp** and **your-address** with the actual filename of your keystore file. 

## Step 4: Configuring Service Node and Off-Chain Bids

Now that the SSL certificate and keystore are in place, it's time to configure your Service Node's parameters and bids for transactions. This involves editing two configuration files: `pantos-service-node.conf` and `pantos-offchain-bids.yaml`.

### Adapt Pantos-Service-Node.conf

You need to review and adapt the Service Node's configuration file appropriately before starting the Service Node instance:

`sudo nano /etc/pantos-service-node.conf`

**In particular, make sure that:**
- the url in the application section matches the domain name of your SSL certificate
- the private_key path of each blockchain points to your keystore file
- the private_key_password is the correct password for your keystore file
- the providers of each blockchain are the blockchain node URLs you want to use for your Service Node
- the correct blockchains have been activated `active: true`

**Example configuration for the Avalanche testnet:**
```
blockchains:
    avalanche:
        active: false
        private_key: /etc/pantos-service-node.keystore
        private_key_password: ADD-YOUR-PASSWORD
        providers:
            - https://api.avax-test.network/ext/bc/C/rpc
        average_block_time: 3
        chain_id: 43113
        hub: '0xdBc273C809473F810A9352716e14E75645528535' #Adjust hub address
        forwarder: '0xc02857899c4198ED17142E66e6B81466581af07F' #Adjust forwarder address
        pan_token: '0xC7895784ca04a41915D916046e304E25c26686dd' #Adjust token address
        confirmations: 20
        min_adaptable_fee_per_gas: 1000000000
        #max_total_fee_per_gas:
        adaptable_fee_increase_factor: 1.101
        blocks_until_resubmission: 20
        stake: 10000000000000
```

### Adapt Pantos-Offchain-Bids.yaml

Each Service Node bid has to be added to the bids list in the configuration section of its source blockchain and contains:
- execution time in seconds
- fee for a Pantos transfer in Panini
- validity period in seconds

By default each blockchain route has two bids already configured, that you can adjust accordingly.

```
bnb_chain:
    ethereum:
      - execution_time: 400
        fee: 25000000000
        valid_period: 300
      - execution_time: 1000
        fee: 9500000000
        valid_period: 300
```
In this example, BNB Chain is the source, and Ethereum is the destination. This configuration activates the ability of your Service Node to handle transactions from BNB Chain to Ethereum.

**After the configuration has been adapted to your liking, start the Service Node instance with:**
```
sudo systemctl start pantos-service-node-server
sudo systemctl start pantos-service-node-celery
```

> **Remember:** You need at least 100.000 PAN and native coins for each blockchain that you want to serve!

Please reach out to the team if you need help getting the required amounts of PAN. 

The log output of the Service Node is then available at
`/var/log/pantos/service-node.log` for the web server and `/var/log/pantos/service-node-worker.log` for the worker process that asynchronously submits transfer requests to the blockchain networks.

If your Pantos Service Node setup is successful, you'll see confirmation in the log files indicating that the node is registered with the Pantos Hub. This means your node is now part of the Pantos network, ready to contribute to its operations.

For any issues during setup, please contact Pantos support for assistance. Thank you for contributing to the Pantos ecosystem!



## OPTIONAL Step 5: Custom Bid Plugins

### Introduction
Service Nodes include a mechanism for bid plugins, allowing you to define custom code to calculate fees for blockchain transfers. This step will guide you through the process of using and customizing these plugins.

### Default Bid Behavior
The service node comes with a default bid plugin that calculates fees based on a YAML configuration file.

**YAML Configuration File Location:** `/etc/pantos-offchain-bids.yaml`

This file contains predefined fees for various blockchain transfer pairs.

### Plugin Files Location
**Base Plugin:** Located at `/opt/pantos/service-node/virtual-environment/lib/python3.10/site-packages/pantos/servicenode/plugins/base.py` 

This file has the abstract BidPlugin class.

**Bids Plugin:** Located at `/opt/pantos/service-node/virtual-environment/lib/python3.10/site-packages/pantos/servicenode/plugins/bids.py`

This file contains the default implementation that reads the YAML file.

### Understanding Bid Plugin Methods

#### 'get_bids' Method:
**Purpose:** This method is the heart of your bid plugin. It's where you define how your service node calculates the fees for transferring tokens between blockchains.

**How it Works:** The method needs to determine fees based on factors like market conditions, transaction complexity, and speed requirements. It returns two things: a list of Bid objects and a time interval in seconds. The Bid objects represent the fee details for each supported blockchain transfer. The time interval tells the service node how often to update these fees.

**Example:** If you want to adjust fees based on current token values or gas price fluctuations, you would code these rules in get_bids.

#### 'accept_bid' Method:

**Purpose:** This method decides whether to accept or reject a transfer request based on the bid provided by the user.

**How it Works:** It examines the bid details submitted by a user wanting to make a transfer. You can write logic to check if the bid meets certain criteria, like minimum fee requirements or if the bid is valid considering current market prices.

**Example:** If there's a sudden spike in a token's value, and the bid no longer covers your operational costs, accept_bid can reject the request.

### Customizing Your Bid Plugin

#### Plan Your Strategy:
Decide how you want your service node to calculate and accept fees. Consider factors like speed, market trends, and operational costs.

#### Create Your Custom Plugin:
- Go to the plugins directory.
- Create a Python file for your plugin.
- Inherit from BidPlugin: Extend the BidPlugin class in your custom file.
- Code your fee calculation and bid acceptance logic in these methods.

#### Activate Your Plugin:
- Update `/etc/pantos-service-node.conf` to point to your new plugin.
- Restart the service node.

Apply your changes by restarting the service node using the command: `sudo systemctl restart pantos-service-node-celery`

#### Verify Operation: 

Check the logs to ensure your plugin is loaded and functioning as expected. Test with various scenarios to confirm the bid calculations are working correctly.

By following these detailed steps, even as a beginner, you can customize the bid plugin on your Pantos Service Node, tailoring it to your specific needs and market strategies.

# Advanced Installation
### Python Wheel package (for advanced users only)

Manually installing a Pantos Service Node instance requires some extra steps to get everything set up as needed. Except for downloading and installing the Debian package, you still need to perform all the steps described in the first section of this guide. Instead of the Debian package, you download the Python Wheel distribution of the Service Node:

`wget https://pantos.io/download/servicenode/wheel`

#### First, install the Service Node's dependencies:
`sudo apt install apache2 apache2-dev iptables libpq-dev postgresql python3-pip python3-venv rabbitmq-server`

Then, disable and stop the Apache 2 web server:
`sudo systemctl disable apache2 && sudo systemctl stop apache2`

#### Add a pantos user:
`sudo adduser --system --group pantos`
And create the Service Node's application directory:
`sudo mkdir -p /opt/pantos/service-node`

#### Next, start the PostgreSQL interactive terminal:
`sudo -u postgres psql`

#### Create the Service Node's database user and databases (replace your_postgres_password with a secure password):
```CREATE ROLE "pantos-service-node" WITH LOGIN PASSWORD 'your_postgres_password';
CREATE DATABASE "pantos-service-node" WITH OWNER "pantos-service-node";
CREATE DATABASE "pantos-service-node-celery" WITH OWNER "pantos-service-node";
\q
```
#### We also need to set up the RabbitMQ message broker (replace your_rabbitmq_password with a secure password):

```
sudo rabbitmqctl add_user pantos-service-node your_rabbitmq_password
sudo rabbitmqctl add_vhost pantos-service-ndoe
sudo rabbitmqctl set_permissions -p pantos-service-node pantos-service-node ".*" ".*" ".*"
```

Now, we need to install the actual Service Node application. For this, move the downloaded Python Wheel file pantos_service_node-VERSION-py3-none-any.whl to /opt/pantos/service-node/ and run these commands (replace VERSION with the actual Service Node version of the downloaded Python Wheel distribution):
```
cd /opt/pantos/service-node/
sudo python3 -m venv virtual-environment
sudo -s
source virtual-environment/bin/activate
python3 -m pip install pantos_service_node-VERSION-py3-none-any.whl
python3 -m pip install mod_wsgi
alembic --config virtual-environment/lib/python3.*/site-packages/pantos/alembic.ini upgrade head
exit
sudo ln -s virtual-environment/lib/python3.*/site-packages/pantos/servicenode/wsgi.py wsgi.py
cd
```

Then, place the configuration file in its proper location and make it accessible to the pantos user:
```
sudo mv /opt/pantos/service-node/virtual-environment/lib/python3.*/site-packages/pantos/pantos-service-node.conf /etc/pantos-service-node.conf
sudo chown pantos:pantos /etc/pantos-service-node.conf
sudo chmod 640 /etc/pantos-service-node.conf
```

#### The Service Node also needs its directory for log files:
```
sudo mkdir -p /var/log/pantos
sudo chown pantos:adm /var/log/pantos/
sudo chmod 750 /var/log/pantos/
```

Afterwards, follow the steps for obtaining an SSL certificate, web server private key, and Ethereum keystore as outlined in the first section of this guide. Follow also the description how to configure your Service Node instance properly. Make sure to add the proper passwords for PostgreSQL and RabbitMQ to your configuration file.

#### Finally, run these commands to start your Service Node instance (replace your.server.name with your actual domain name):

```
sudo -u pantos bash -c "source /opt/pantos/service-node/virtual-environment/bin/activate; nohup mod_wsgi-express start-server --https-port 8443 --https-only --server-name your.server.name --ssl-certificate-file /etc/pantos-service-node-fullchain.pem --ssl-certificate-key-file /etc/pantos-service-node-privkey.pem /opt/pantos/service-node/wsgi.py >> /var/log/pantos/service-node-mod_wsgi.log 2>&1 &"
sudo -u pantos bash -c "source /opt/pantos/service-node/virtual-environment/bin/activate; nohup celery -A pantos.servicenode worker -l INFO -n pantos.servicenode -Q pantos.servicenode >> /var/log/pantos/service-node-celery.log 2>&1 &"
iptables-nft -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 8443
```
You may also add these commands to a startup script for more conveniently starting the Service Node.
