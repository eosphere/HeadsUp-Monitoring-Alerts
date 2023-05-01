# Architechure

A running HeadsUp implementation consists of 3 components Website, Ingestor and Database.

HeadsUp is designed for Antelope Node Operators to run a private instance of the platform on their internal network using docker for each of the components and docker compose to orchestrate and configure the base deployment.

## Website

The Website is the central hub of the HeadsUp platform. The website receives metrics and alerts from the Ingestor (over a websocket) and forwards them onto whoever needs to receive them. This can be the website frontend, email or slack recipients. 

The website is a SPA (single page application) when the user browses to it, it opens a websocket to the website. The website then sends metrics and alerts down that websocket.

## Ingestor
The Ingestor is the engine of the HeadsUp platform. The Ingestor is resposible for querying all monitored nodes and providing structured metric data to the Website.

## Database

A postgres database which holds the current configuration information and a list of recent alerts. By default we run this in a docker container and use a docker volume to ensure the database is persisted.

## Hardware and Software Requirements
- 4 Core CPU
- 64GB Disk
- 2GB RAM
- Ubuntu 22.04 **_(Recommended)_**
- docker
- docker compose

# Setting Up

This guide follows installing HeadsUp on a server running Ubuntu 22.04 however other docker compatible distributions will work with minor adjustments. The steps to a running production HeadsUp instance are the following:

1) Install docker and docker compose
2) Configure the docker-compose.yaml
3) Initial startup
4) Configure Chains
5) Configure Nodes
6) Configure Metrics
7) Set Alerts

## 1) Install docker and docker compose

**docker**
```
> sudo apt update

> sudo apt install apt-transport-https ca-certificates curl software-properties-common

> curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

> echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

> sudo apt update

*Check what version will be installed*
> apt-cache policy docker-ce

> sudo apt install docker-ce

*Check docker version*
> docker --version
```
**NB:** Docker requires that the user that will be running docker is in the docker user group, assuming you don't want to run HeadsUp as root add your user to the docker group as below:
```
> sudo usermod -aG docker ${USER}
```

**docker compose**
```
> mkdir -p ~/.docker/cli-plugins/

> curl -SL https://github.com/docker/compose/releases/download/v2.3.3/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose

> chmod +x ~/.docker/cli-plugins/docker-compose

*Check docker compose version*
> docker compose version
```
## 2) Configure the docker-compose.yaml
Download the latest HeadsUp example `docker-compose.yaml` and configure neccessary fields with your own details as below:
```
> wget https://github.com/eosphere/HeadsUp-Monitoring-Alerts/blob/main/docker-compose.yaml

> nano docker-compose.yaml

website:
  environment:
    - headsup_server_masterpassword=<ENTER NEW PASSWORD>
    - headsup_server_jwtsecret=<ENTER NEW RANDOM SEED>
    - headsup_server_postgrespassword=<POSTGRES PASSWORD>
    - headsup_server_rooturl=<https://YOUR ROOT URL>

#Alerts Email and Slack Webhook - EOSphere Example#
    - headsup_server_emailalerts=true
    - headsup_server_emailsender=alerts@eosphere.io
    - headsup_server_emailto=ross@eosphere.io,shelley@eosphere.io,john@eosphere.io
    - headsup_server_emailsmtpserver=smtp.gmail.com
    - headsup_server_emailsmtpssl=true
    - headsup_server_emailsmtptls=false
    - headsup_server_emailsmtpport=465
    - headsup_server_emailsmtpuser=alerts@eosphere.io
    - headsup_server_emailsmtppassword=E6oNZPfZ6r5g
    - headsup_server_slackalerts=true
    - headsup_server_slackhook=https://hooks.slack.com/services/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o

postgres:
  environment:
    - POSTGRES_PASSWORD=<POSTGRES PASSWORD>

```

**headsup_server_masterpassword** is the master password for the single user who has the authority to configure HeadsUp. Keep it a secret to prevent others making malicious changes to your server. This is a web user login, Headsup does not maintain a list of user and password pairs there is only one user with one password.

**headsup_server_jwtsecret** is the random string of characters used to generate a private key to sign authentication tokens. The most secure way to generate this is to use a password generator. If the master password above leaks you should change this value in addition to the master password.

**headsup_server_debugingestorpassword** should not be set for production systems.

**headsup_server_emailalerts** Set to true if you want to send email alerts.

**headsup_server_emailsender** The from address for email alerts

**headsup_server_emailto** The address to send alerts to

**headsup_server_emailsmtpserver** The address of the SMTP server to send the emails to

**headsup_server_emailsmtpssl** Use ssl for the connection to the email server

**headsup_server_emailsmtptls** Use tls for the connection to the email server

**headsup_server_emailsmtpport** Port to connect to the SMTP server on (remember to adjust if using SSL or TLS)

**headsup_server_emailsmtpuser** User name to use when connecting to the SMTP server

**headsup_server_emailsmtppassword** Password to use when connecting to the SMTP server

**headsup_server_rooturl** The root of your webserver, used for calculating the full URLs for silencing (eg https://headsup.eosphere.io)

**headsup_server_slackalerts** Set to true if want to send slack alerts

**headsup_server_slackhook** The slack hook URL for your postgres (https://slack.com/intl/en-au/help/articles/115005265063-Incoming-webhooks-for-Slack)

### 2.1) Upgrading to use ingestor_aux

Version v0.0.129 added a new program to the docker-compose configure, ingestor_aux. This is an additional program which takes some of the load off the ingestor. You can upgrade your docker-compose.yaml file from an earlier version by doing the following.

1) Upgrade the image version to v0.0.129 for all the container.
2) Copy the ingestor-aux section from the example docker-compose.
3) Change the ingestor section so that the ingestor depends on ingestor_aux.
4) Restart with `docker-compose up -d`

## 3) Initial Startup

HeadsUp requires a once-off manual sequenced start of it's containers during initial startup

**First the postgres Database container**

```
> docker compose up postgres
```
This will download the postgres container and then run it which initialises the database. We do this because of the time it takes to initialise this container. When the website container starts it tries to run the database migrations immediately, if the database is not ready this causes an issue.

Once the database container is initialised press Ctrl+C to stop it.

**Next start the Website container**

```
> docker compose up website
```
Once it has started **Access the HeadsUp Website** through a web browser (http://<SERVER_IP_ADDRESS>:80)

**Login** with the master password by clicking the lock on the top right.

Click the **Setup** on the left hand side

Click the **Plus/+** sign next to ingestors

Give your **Ingestor a Name** (Typically Ingestor 1) and set the **Sort Order** to 0

Click the **Save** and once it has saved click the **Ingestor Access Key** arrow and lock button

**Copy the Code** it displays and Click the **Set** button to update the server

_The access key is similar to a password for the Ingestor and the name is similar to a username_

Edit the `docker-config.yaml` file to place the new access_key in the headsup_accesskey in the ingestor section
```
> nano docker-compose.yaml
services:
  ingestor:
  environment:
    - headsup_accesskey=<ENTER ACCESS KEY HERE>
```
Press Ctrl+C to stop the Website container.

The manual initialisation process is complete, press Ctrl+C to stop the Website container.

**Next start HeadsUp with docker compose**
```
> docker compose up -d
```
This will orchestrate the launch of the HeadsUp platform and will run docker compose in the background. 

You can check running docker processes with: 
```
> docker ps
or
> docker stats --all
```
In this configuration docker compose will automatically restart the HeadsUp platform on reboot.

## 4) Configure Chains
HeadsUp has the ability to monitor and alert on multiple Antelope blockchains at the same time, each new chain is essentially a discrete blockchain network Mainnet/Testnet etc.

Connect to the website and **Login** as the master user.

Click **Home**

You should see a blank screen as you have not set up any chains.

Click **+ Add Chain**

Configure the following fields:

**Chain Name**
A human readable name for the chain. This name is the name used to identify the chain in the HeadsUp's user interface.

**Chain Type**
You can choose the chain type, this will prefillout some of the values (chain id and Vote Decay Type) with that chain's known settings and use the correct chain icon. Unknown chains can use the `Private Chain` type.

**Icon**
This is the icon to use for this chain, you cannot customise this setting.

**BP Name**
For metrics which monitor your Block Producer this is the name of your on-chain Block Producer Account.

**Vote Decay Type**
The original EOS mainnet halved the values of votes after 52 weeks, WAX uses 13 weeks and Telos doesn't have vote decay any more at all. You can only use one of the preset options.

**Chain Id**
The Antelope Chain Id. Each Mainnet and Testnet (and Private Network) will have a unique one of these.

**Sort Order**
The order in which to display this chain in the on-screen list.

Click **Create >** and the new chain will be added to the left hand panel.

## 5) Configure Nodes
Nodes are configured within each chain, they monitor specific nodeos, atomic or hyperion API's. As HeadsUp is designed to be run privately, for best operation a node should be configured to directly query the API rather than a loadbalancer or any other intermediatry.

Connect to the website and **Login** as the master user.

Click **Home**

Click on the relevent **Chain**

Click **Add Node**

Configure the following fields:

**Node Name**
A human readable name for the node. This name is the name used to identify the node in the HeadsUp's user interface.

**Node Type**
Available types
Block Producer - The primary Nodeos Block Producer Node
Nodeos Follower - The secondary Nodeos instances that measure their block sync status against the Block Producer Headblock
Hyperion - The Hyperion API
Atomic Assets - The Atomic Assets API
Website - Monitor and alert on your website, chains.json and bp.json files

The difference between "Block Producer" and "Nodeos Follower" in the headblock number metric. For the purposes of the alert the headblock number of the follower node in compared to the headblock number of the Block Producer Node.

**IP Address**
Address of the node, can also be a DNS name

**Port**
Port to connect to the node on

**Sort Order**
On screen display order of the nodes in Headsup

**Use SSL for connections**
Connect via HTTPS otherwise unencrypted HTTP

**Ingestor**
It is possible to have multiple ingestors although at this point we only use `Ingestor 1`

Click **Create >**

_A connect metric will need to be configured before a new node shows as reachable_

## 6) Configure Metrics

Metrics define what is displayed on the HeadsUp dashboard as well as make various alerts available. 

Connect to the website and **Login** as the master user.

Click **Home**

Click on the relevent **Chain**

Click on the relevent **Node**

Click the **Bell Icon** in the top right hand corner

Metrics are devided into **Metric Modules** of related values. All value derived directly from a Nodeos, Hyperion or Atomic Node are in their own groups inside which numerous metrics are selectable.

Choose your **Metric Module** by opening the drop down box and selecting the relevent module.
Choose your **Metrics** by opening the drop down box and ticking the relevent metric.

### Available Metrics

---

**Node/Connects**
Is the node reachable

**Node/Latency**
Time in milliseconds to connect to the node

**Node/Head_Block**
The head block number

**Node/Chain_Id**
Chain Id the node reports

**Node/Last_Irreversible_Block**
The node reported last irreversible block

**Node/Server_Version**
The node reported software version

**Node/Head_Block_Producer**
The node reported current head block producer

**Node/Supported_Apis_Advertised**
A list of the support APIs

**Node/Db_Size**
Reported size of the chain state database

**Node/Db_Size_Free**
Reported free space in the chain state database

**Node/Db_Size_Used**
Reported used space in the chain state database

**Node/Claimer_Has_Run**
Have BP rewards been claimed in the last 24 hours

---

**History_Api/Connects**
Is the legacy History API accepting connections

---

**Vote/Total**
Total number of votes for the configured Block Producer account

**Vote/Rank**
Position in the vote rankings for the configured Block Producer account

**Vote/Percentage**
Percentage of total votes for the configured Block Producer account

**Vote/ActiveProducerHasProduced**
If the configured Block Producer Account is in the top 21 has it produced when it should have. This will alert if the block producer is in the top 21 but hasn't produced.

---

**Website/ChainsJsonAndBpJson**
Are the chains.json and bp.json files set up correctly.

---

**Hyperion/Connects**
Is the Hyperion API reachable

**Hyperion/Version**
The Hyperion reported software version

**Hyperion/Features**
List of available features reported by the Hyperion Software

**Hyperion/Latency**
Time in milliseconds to connect to the node

**Hyperion/NodeosRpc**
Is the Nodeos RPC service reported in good health by Hyperion

**Hyperion/ElasticSearchStatus**
Is the Elasticsearch service reported in good health by Hyperion

**Hyperion/ElasticSearchIndexedBlock**
Last reported block indexed by Elasticsearch

**Hyperion/BlockNumber**
Last nodeos block reported to Hyperion

---

**AtomicAssets/Latency**
Time in milliseconds to connect to the node

**AtomicAssets/Connects**
Is the Atomic API reachable

**AtomicAssets/Version**
The Atomic API reported software version

**AtomicAssets/Status**
Status as reported by Atomic API

**AtomicAssets/BlockNumber**
The last reported block number processed by the node

## 7) Set Alerts

Metrics only advise on a value while alerts will send a message when a certain condition is met. Depending on your configuration these alerts notify via the website dashboard, email and/or slack.

Alerts are set on each node and are available when a corresponding metric has been selected in the same Metrics and Alert dialog box.

Connect to the website and **Login** as the master user.

Click **Home**

Click on the relevent **Chain**

Click on the relevent **Node**

Click the **Bell Icon** in the top right hand corner

Click **Alerts**

Each alert has a condition field which is what it will test against, there is an explanation of the condition field on the right hand side.

### New User Alerts Recommendation
Start with experiment setting thresholds for alerts relevent to the Node Type, some Metrics will automatically be alerted on when selected such as Node/Connects or Node/Claimer_Has_Run

These alerts are quite handy to start with:
- Node/Db_Size_Free : 4096
- Node/HeadBlock : 100
- Node/Latency : 2000

## Useful commands while running Headsup

### Viewing logs

`journalctl -u docker.service -n 25`

Show the last 25 log entries.

`journalctl -f -u docker.service`

Stream logs to the console as they occur. Useful for detemining the liveness of headsup

### Cleaning up old containers

```
docker images | grep headsup-website | tr -s ' ' | cut -d ' ' -f 2 | xargs -I {} docker rmi eosphere/headsup-website:{}
docker images | grep headsup-ingestor | tr -s ' ' | cut -d ' ' -f 2 | xargs -I {} docker rmi eosphere/headsup-ingestor:{}
docker images | grep headsup-ingestor-aux | tr -s ' ' | cut -d ' ' -f 2 | xargs -I {} docker rmi eosphere/headsup-ingestor-aux:{}
```

This will delete all container images with headsup-website and headsup-ingestor in them, except for the running containers. So as long as docker compose is running these will clean up old container images. The container images are qute large and useless once upgraded so you probably want to do this as part of your usual maintenace after performing an upgrade.

