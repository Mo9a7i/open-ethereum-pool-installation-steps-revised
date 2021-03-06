## Open Source Callisto (CLO) Mining Pool

### Features  

**This is an updated version of installation instructions of open-ethereum-pool and its forks, you can adapt to your coin!**

* Support for HTTP and Stratum mining
* Detailed block stats with luck percentage and full reward
* Failover geth instances: geth high availability built in
* Modern beautiful Ember.js frontend
* Separate stats for workers: can highlight timed-out workers so miners can perform maintenance of rigs
* JSON-API for stats
* PPLNS block reward
* Multi-tx payout at once
* Beautiful front-end highcharts embedded

#### Proxies

* [Ether-Proxy](https://github.com/sammy007/ether-proxy) HTTP proxy with web interface
* [Stratum Proxy](https://github.com/Atrides/eth-proxy) for Ethereum

## Guide to make your very own Eth-based mining pool

### Building on Linux

Dependencies:

  * go >= 1.10
  * redis-server >= 2.8.0
  * nodejs >= 4 LTS
  * nginx
  * geth (multi-geth)

**I highly recommend to use Ubuntu 16.04 LTS.**

### add a user for the pool coinbase

    $ adduser clopool
    $ usermod -aG sudo clopool
    $ su clopool
    $ cd ~

### Install the required Dependencies

    $ sudo apt-get update -y
    $ sudo apt-get install -y software-properties-common build-essential unzip curl git nano
    $ sudo apt-get upgrade -y

### Remove unnecessary services
    $ sudo apt-get -y autoremove apache2


### Install go lang

    $ sudo add-apt-repository -y ppa:gophers/archive
    $ sudo apt-get -y update
    $ sudo apt-get -y install golang-1.10-go
    $ sudo ln -s /usr/lib/go-1.10/bin/go /usr/local/bin/go

### Install redis-server

    $ sudo apt-get -y install redis-server

It is recommended to bind your DB address on 127.0.0.1 or on internal ip. Also, please set up the password for advanced security!!!

#### Bind to 127.0.0.1

    $ sudo nano /etc/redis/redis.conf

Uncomment `bind 127.0.0.1`

#### Setup a Password

    $ echo "password" | sha256sum

copy the generated password and paste in the following

    $ sudo nano /etc/redis/redis.conf

uncomment this line  `requirepass foobared`
replace foobared with the generated password you copied.

    $sudo service redis-server restart

To check that Redis is working, use the Redis command line.

    $ redis-cli

to authenticate, do the following in **redis-cli** 'auth yourHashedString'

### Install nginx

    $ sudo apt-get install nginx

Search on Google for nginx-setting

### Install NODEJS

This will install the latest nodejs

    $ sudo curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
    $ sudo apt-get install -y nodejs

### Install multi-geth
*MAke sure to compile the appropriate version of `geth` for the coin you're pool is for*

    $ sudo wget https://github.com/ethereumsocial/multi-geth/releases/download/v1.8.10/multi-geth-linux-v1.8.10.zip
    $ sudo unzip multi-geth-linux-v1.8.10.zip
    $ sudo mv geth /usr/local/bin/geth

### Run multi-geth

If you use Ubuntu, it is easier to control services by using serviced.

    $ sudo nano /etc/systemd/system/callisto.service

Copy the following example

```
[Unit]
Description=Callisto for Pool
After=network-online.target

[Service]
ExecStart=/usr/local/bin/geth --callisto --cache=1024 --extradata "Mined by <your-pool-domain>" --ethstats "<your-pool-domain>:Callisto@clostats.net" --unlock="Wallet_Address" --password="/home/<your-user-name>/.walletpass" --rpc --rpcaddr 127.0.0.1 --rpcapi eth,net,web3
User=<your-user-name>

[Install]
WantedBy=multi-user.target
```

Then run multi-geth by the following commands

    $ sudo systemctl enable callisto
    $ sudo systemctl start callisto

If you want to debug the node command

    $ sudo systemctl status callisto

To monitor synchronization progress

    $ sudo journalctl -f -u callisto

Run console

    $ geth attach

*You can create your own shortcut for `geth attach` on `/usr/local/bin/gethattach` for example, make sure to include* **--datadir** *for the ipc*

    $ sudo nano /usr/local/bin/attachgeth.sh

````
/usr/local/bin/geth attach ipc://home/<your-user-name>/.ethereum/callisto/geth.ipc
````

    $ chmod 755 /usr/local/bin/attachgeth.sh

Now you can use `atachgeth.sh` from anywhere in your servers.




Register pool account and open wallet for transaction. This process is always required, when the wallet node is restarted.

    > personal.newAccount()
    > personal.unlockAccount(eth.accounts[0],"password",40000000)
    > exit

#### Save wallet password
Save wallet password in `/home/pooluser/.walletpass` and embed it in the pool.service line

### Install Callisto Pool

    $ sudo git clone https://github.com/chainkorea/open-callisto-pool
    $ sudo cd open-callisto-pool
    $ sudo make all

If you get a `open-callisto-pool` after ls `ls ~/open-callisto-pool/build/bin/`, the installation has completed.



### Set up Callisto pool

Here, you can use [phatblinkie's](https://github.com/phatblinkie/cryptopools.info-pool/blob/master/service_installer.sh) service_installer.sh (*if you know how to use it*)

Or, do the regular:

    $ mv config.example.json config.json
    $ nano config.json

Set up based on commands below.

```javascript
{
  // The number of cores of CPU.
  "threads": 2,
  // Prefix for keys in redis store
  "coin": "clo",
  // Give unique name to each instance
  "name": "main",
  // PPLNS rounds
  "pplns": 9000,

  "proxy": {
    "enabled": true,

    // Bind HTTP mining endpoint to this IP:PORT
    "listen": "0.0.0.0:8888",

    // Allow only this header and body size of HTTP request from miners
    "limitHeadersSize": 1024,
    "limitBodySize": 256,

    /* Set to true if you are behind CloudFlare (not recommended) or behind http-reverse
      proxy to enable IP detection from X-Forwarded-For header.
      Advanced users only. It's tricky to make it right and secure.
    */
    "behindReverseProxy": false,

    // Stratum mining endpoint
    "stratum": {
      "enabled": true,
      // Bind stratum mining socket to this IP:PORT
      "listen": "0.0.0.0:8008",
      "timeout": "120s",
      "maxConn": 8192
    },

    // Try to get new job from geth in this interval
    "blockRefreshInterval": "120ms",
    "stateUpdateInterval": "3s",
    // If there are many rejects because of heavy hash, difficulty should be increased properly.
    "difficulty": 2000000000,

    /* Reply error to miner instead of job if redis is unavailable.
      Should save electricity to miners if pool is sick and they didn't set up failovers.
    */
    "healthCheck": true,
    // Mark pool sick after this number of redis failures.
    "maxFails": 100,
    // TTL for workers stats, usually should be equal to large hashrate window from API section
    "hashrateExpiration": "3h",

    "policy": {
      "workers": 8,
      "resetInterval": "60m",
      "refreshInterval": "1m",

      "banning": {
        "enabled": false,
        /* Name of ipset for banning.
        Check http://ipset.netfilter.org/ documentation.
        */
        "ipset": "blacklist",
        // Remove ban after this amount of time
        "timeout": 1800,
        // Percent of invalid shares from all shares to ban miner
        "invalidPercent": 30,
        // Check after after miner submitted this number of shares
        "checkThreshold": 30,
        // Bad miner after this number of malformed requests
        "malformedLimit": 5
      },
      // Connection rate limit
      "limits": {
        "enabled": false,
        // Number of initial connections
        "limit": 30,
        "grace": "5m",
        // Increase allowed number of connections on each valid share
        "limitJump": 10
      }
    }
  },

  // Provides JSON data for frontend which is static website
  "api": {
    "enabled": true,
    "listen": "0.0.0.0:8080",
    // Collect miners stats (hashrate, ...) in this interval
    "statsCollectInterval": "5s",
    // Purge stale stats interval
    "purgeInterval": "10m",
    // Fast hashrate estimation window for each miner from it's shares
    "hashrateWindow": "30m",
    // Long and precise hashrate from shares, 3h is cool, keep it
    "hashrateLargeWindow": "3h",
    // Collect stats for shares/diff ratio for this number of blocks
    "luckWindow": [64, 128, 256],
    // Max number of payments to display in frontend
    "payments": 50,
    // Max numbers of blocks to display in frontend
    "blocks": 50,
    // Frontend Chart related settings
    "poolCharts":"0 */20 * * * *",
    "poolChartsNum":74,
    "minerCharts":"0 */20 * * * *",
    "minerChartsNum":74

    /* If you are running API node on a different server where this module
      is reading data from redis writeable slave, you must run an api instance with this option enabled in order to purge hashrate stats from main redis node.
      Only redis writeable slave will work properly if you are distributing using redis slaves.
      Very advanced. Usually all modules should share same redis instance.
    */
    "purgeOnly": false
  },

  // Check health of each geth node in this interval
  "upstreamCheckInterval": "5s",

  /* List of geth nodes to poll for new jobs. Pool will try to get work from
    first alive one and check in background for failed to back up.
    Current block template of the pool is always cached in RAM indeed.
  */
  "upstream": [
    {
      "name": "main",
      "url": "http://127.0.0.1:8545",
      "timeout": "10s"
    },
    {
      "name": "backup",
      "url": "http://127.0.0.2:8545",
      "timeout": "10s"
    }
  ],

  // This is standard redis connection options
  "redis": {
    // Where your redis instance is listening for commands
    "endpoint": "127.0.0.1:6379",
    "poolSize": 10,
    "database": 0,
    "password": ""
  },

  // This module periodically remits ether to miners
  "unlocker": {
    "enabled": false,
    // Pool fee percentage
    "poolFee": 1.0,
    // the address is for pool fee. Personal wallet is recommended to prevent from server hacking.
    "poolFeeAddress": "",
    // Amount of donation to a pool maker. 10 percent of pool fee is donated to a pool maker now. If pool fee is 1 percent, 0.1 percent which is 10 percent of pool fee should be donated to a pool maker.
    "donate": true,
    // Unlock only if this number of blocks mined back
    "depth": 120,
    // Simply don't touch this option
    "immatureDepth": 20,
    // Keep mined transaction fees as pool fees
    "keepTxFees": false,
    // Run unlocker in this interval
    "interval": "10m",
    // Geth instance node rpc endpoint for unlocking blocks
    "daemon": "http://127.0.0.1:8545",
    // Rise error if can't reach geth in this amount of time
    "timeout": "10s"
  },

  // Pay out miners using this module
  "payouts": {
    "enabled": true,
    // Require minimum number of peers on node
    "requirePeers": 5,
    // Run payouts in this interval
    "interval": "12h",
    // Geth instance node rpc endpoint for payouts processing
    "daemon": "http://127.0.0.1:8545",
    // Rise error if can't reach geth in this amount of time
    "timeout": "10s",
    // Address with pool coinbase wallet address.
    "address": "0x0",
    // Let geth to determine gas and gasPrice
    "autoGas": true,
    // Gas amount and price for payout tx (advanced users only)
    "gas": "21000",
    "gasPrice": "50000000000",
    // The minimum distribution of mining reward. It is 1 CLO now.
    "threshold": 1000000000,
    // Perform BGSAVE on Redis after successful payouts session
    "bgsave": false
    "concurrentTx": 10
  }
}
```
**This file has settings for, API, 1x Stratum, Payout, Unlocker, all combined together to run as one service**
  * Stratum is the node miners will connect two with a set difficulty
  * API is the service that submits/reports stats to the website
  * Unlocker is the service that does the accounting based on the payout scheme, which miner gets how much of the block, and writes down to DB
  * Payout is the service that does the actual transactions to miners.


If you are distributing your pool deployment to several servers or processes,
create several configs and disable unneeded modules on each server. (Advanced users)

I recommend this deployment strategy:

* Mining instance - 1x (it depends, you can run one node for EU, one for US, one for Asia)
* Unlocker and payouts instance - 1x each (strict!)
* API instance - 1x


### Run Pool
It is required to run pool by serviced. If it is not, the terminal could be stopped, and pool doesn’t work.

    $ sudo nano /etc/systemd/system/clopool.service

Copy the following example

```
[Unit]
Description=Callisto
After=callisto.target

[Service]
Type=simple
ExecStart=/home/clopool/open-callisto-pool/build/bin/open-callisto-pool /home/clopool/open-callisto-pool/config.json

[Install]
WantedBy=multi-user.target
```

Then run pool by the following commands

    $ sudo systemctl enable clopool
    $ sudo systemctl start clopool

If you want to debug the node command

    $ sudo systemctl status clopool

Backend operation has completed so far.
**If you want to separate each service of the pool in a separate process, then you need to make different configs and a service for each**

### Open Firewall

Firewall should be opened to operate this service. Whether Ubuntu firewall is basically opened or not, the firewall should be opened based on your situation.
You can open firewall by opening 80,443,8080,8888,8008.

## Install Frontend

### Modify configuration file

    $ nano ~/open-callisto-pool/www/config/environment.js

Make some modifications in these settings.

    BrowserTitle: 'Callisto Mining Pool',
    ApiUrl: '//your-pool-domain/',
    HttpHost: 'http://your-pool-domain',
    StratumHost: 'your-pool-domain',
    PoolFee: '1%',

The frontend is a single-page Ember.js application that polls the pool API to render miner stats.

    $ cd ~/open-callisto-pool/www
    $ sudo npm install -g ember-cli@2.9.1
    $ sudo npm install -g bower
    $ sudo chown -R $USER:$GROUP ~/.npm
    $ sudo chown -R $USER:$GROUP ~/.config
    $ npm install
    $ bower install
    $ ./build.sh
    $ cp -R ~/open-callisto-pool/www/dist ~/www

As you can see above, the frontend of the pool homepage is created. Then, moved to the directory `www` which serves the file.
*Change `favicon.ico` file to your taste (act special)*


## Set up Nginx.

    $ sudo nano /etc/nginx/sites-available/default

Modify based on configuration file.

````
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /home/<your-user-name>/www;

        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                try_files $uri $uri/ =404;
        }

        location /api {
                proxy_pass http://127.0.0.1:8080/api;
          }
    }
````

After setting nginx is completed, run the command below.

    $ sudo service nginx restart

Type your homepage address or IP address on the web.
If you face screen without any issues, pool installation has completed.

### Extra) How To Secure the pool frontend with Let's Encrypt (https)

This guide was originally referred from [digitalocean - How To Secure Nginx with Let's Encrypt on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04)

First, install the Certbot's Nginx package with apt-get

```
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install python-certbot-nginx
```

And then open your nginx setting file, make sure the server name is configured!

```
$ sudo nano /etc/nginx/sites-available/default
. . .
server_name <your-pool-domain>;
. . .
```

Change the _ to your pool domain, and now you can obtain your auto-renewed SSL certificate for free!

```
$ sudo certbot --nginx -d <your-pool-domain>
```

Now you can access your pool's frontend via https! Share your pool link!

### Notes

* Unlocking and payouts are sequential, 1st tx go, 2nd waiting for 1st to confirm and so on. You can disable that in code. Carefully read `docs/PAYOUTS.md`.
* Also, keep in mind that **unlocking and payouts will halt in case of backend or node RPC errors**. In that case check everything and restart.
* You must restart module if you see errors with the word *suspended*.
* Don't run payouts and unlocker modules as part of mining node. Create separate configs for both, launch independently and make sure you have a single instance of each module running.
* If `poolFeeAddress` is not specified all pool profit will remain on coinbase address. If it specified, make sure to periodically send some dust back required for payments.
* DO NOT OPEN YOUR RPC OR REDIS ON 0.0.0.0!!! It will eventually cause coin theft.

### Credits

Made by sammy007. Licensed under GPLv3.
Modified by Akira Takizawa & The Ellaism Project.

#### Updated / Edited By:
* **Brian Bowen** - [phatblinkie](https://github.com/phatblinkie)
* **Mohannad Otaibi** - [mo9a7i](https://github.com/mo9a7i)

#### Contributors

[Alex Leverington](https://github.com/subtly)

### Donations

ETH/ETC/ETSC/CLO: 0x34AE12692BD4567A27e3E86411b58Ea6954BA773

![](https://cdn.pbrd.co/images/GP5tI1D.png)

Highly appreciated.
