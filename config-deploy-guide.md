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
- docker compose v2.27.0 **_(Recommended)_**

# Setting Up

This guide follows installing HeadsUp on a server running Ubuntu 22.04 however other docker compatible distributions will work with minor adjustments. The steps to a running production HeadsUp instance are the following:

1) Install docker compose and associated applications
2) Configure the docker-compose.yaml
3) Initial startup
4) Configure Chains
5) Configure Nodes
6) Configure Metrics
7) Set Alerts


## 1) Install docker compose and associated applications

**Add Docker's official GPG key:**
```
sudo apt-get update

sudo apt-get install ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```
**Add the repository to Apt sources:**

```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```
**Install:**
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
**Check docker compose version:**
```
docker compose version

Docker Compose version v2.27.0
```
**NB:** Docker requires that the user that will be running docker is in the docker user group, assuming you don't want to run HeadsUp as root add your user to the docker group as below:
**Add user to the docker group:**
```
sudo usermod -aG docker ${USER}
```

## 2) Configure the docker-compose.yaml
Download the latest HeadsUp example `docker-compose.yaml` and configure neccessary fields with your own details as below in this EOSphere example:
```
wget https://raw.githubusercontent.com/eosphere/HeadsUp-Monitoring-Alerts/main/example-docker-compose.yaml

cp example-docker-compose.yaml docker-compose.yaml

nano docker-compose.yaml

website:
  environment:
    - headsup_server_masterpassword=headsup123pass
    - headsup_server_jwtsecret=NqPFk9icjklswHgKZZ5J
    - headsup_server_postgrespassword=postgres123
    - headsup_server_rooturl=https://headsup.eosphere.io
    - headsup_server_logduration=7
    - headsup_server_failedpingsthreshold=3
    - headsup_server_reminderfrequency=15

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
    - headsup_server_telegramalerts=true                                                                                                                                                                                            
    - headsup_server_telegrambot=7867365320:ABHkXyeDb7eB-ZiplTx-5BCcChQqE9FiJMg                                                                                                                                                     
    - headsup_server_telegramchat=-4379740104  
    - headsup_server_slackalerts=true
    - headsup_server_slackhook=https://hooks.slack.com/services/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o

postgres:
  environment:
    - POSTGRES_PASSWORD=postgres123

```

**headsup_server_masterpassword** is the master password for the single user who has the authority to configure HeadsUp. Keep it a secret to prevent others making malicious changes to your server. This is a web user login, Headsup does not maintain a list of user and password pairs there is only one user with one password.

**headsup_server_jwtsecret** is the random string of characters used to generate a private key to sign authentication tokens. The most secure way to generate this is to use a password generator. If the master password above leaks you should change this value in addition to the master password.

**headsup_server_rooturl** The root of your webserver, used for calculating the full URLs for silencing (eg https://headsup.eosphere.io)

**headsup_server_debugingestorpassword** should not be set for production systems.

**headsup_server_logduration** the number of days to keep resolved alerts in history. 

**headsup_server_failedpingsthreshold** the number of consecutive failed conditions to be met before alerting.

**headsup_server_reminderfrequency** the frequency in minutes to remind of an ongoing alert.

**headsup_server_emailalerts** Set to true if you want to send email alerts.

**headsup_server_emailsender** The from address for email alerts

**headsup_server_emailto** The address to send alerts to

**headsup_server_emailsmtpserver** The address of the SMTP server to send the emails to

**headsup_server_emailsmtpssl** Use ssl for the connection to the email server

**headsup_server_emailsmtptls** Use tls for the connection to the email server

**headsup_server_emailsmtpport** Port to connect to the SMTP server on (remember to adjust if using SSL or TLS)

**headsup_server_emailsmtpuser** User name to use when connecting to the SMTP server

**headsup_server_emailsmtppassword** Password to use when connecting to the SMTP server

**headsup_server_telegrambot** Telegram bot token

**headsup_server_telegramchat** Telegram bot chat ID

**headsup_server_slackalerts** Set to true if want to send slack alerts

**headsup_server_slackhook** The slack hook URL for your postgres (https://slack.com/intl/en-au/help/articles/115005265063-Incoming-webhooks-for-Slack)


## 3) Initial Startup

HeadsUp requires a once-off manual sequenced start of it's containers during initial startup

**First the postgres Database container**

```
docker compose up postgres
```
This will download the postgres container and then run it which initialises the database. We do this because of the time it takes to initialise this container. When the website container starts it tries to run the database migrations immediately, if the database is not ready this causes an issue.

Once the database container is initialised press Ctrl+C to stop it.

**Next start the Website container to create and add an Access Key**

```
docker compose up website
```
Once it has started **Access the HeadsUp Website** through a web browser (http://<SERVER_IP_ADDRESS>:80)

**Login** with the master password by clicking the lock on the top right.

Click the **Setup** on the left hand side

Click the **Plus/+** sign next to ingestors

Give your **Ingestor a Name** (Typically Ingestor 1) and set the **Sort Order** to 0

Click the **Save** button 

Once it has saved click the **Ingestor Key and Plus/+ icon** to "generate new key"

Click **Generate Key**

Click **Copy** to copy the Access Key and then click **Done**

_The access key is similar to a password for the Ingestor and the name is similar to a username_

Optionally give your dashboard an **instance_title** to be displayed in the banner and tab.

Optionally add your **organisation_domain** to be used for ```chains.json``` and ```bp.json``` monitoring.

Press Ctrl+C to stop the Website container.

Edit your `docker-config.yaml` file to place the new access_key in the headsup_accesskey in the ingestor section
```
nano docker-compose.yaml

services:
  ingestor:
  environment:
    - headsup_name=Ingestor 1    
    - headsup_accesskey=<ENTER ACCESS KEY HERE>
```

The manual initialisation process is complete.

**Next start HeadsUp with docker compose**
```
docker compose up -d
```
This will orchestrate the launch of the HeadsUp platform and will run docker compose in the background. 

You can check running docker processes with: 
```
docker ps

or

docker stats --all
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

**Monitor BP.json for the chain**
Select if you would like to monitor the ```chains.json``` -> ```bp.json``` for this chain advertised under the **organisation_domain**.

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


### Configure the following fields:

**Node Name**

A human readable name for the node. This name is the name used to identify the node in the HeadsUp user interface.

**Node Type**

Block Producer - The primary Nodeos Block Producer Node

Nodeos Follower - The secondary Nodeos instances that measure their block sync status against the Block Producer Headblock

Hyperion - The Hyperion API

Atomic Assets - The Atomic Assets API

Website - Monitor and alert on your website, chains.json and bp.json files

The difference between "Block Producer" and "Nodeos Follower" is the headblock number metric. For the purposes of the alert the headblock number of the follower node in compared to the headblock number of the Block Producer Node.

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

_A Default set of Metrics and Alerts will be configured upon creation of each node_


## 6) Configure Metrics

Metrics define what is displayed on the HeadsUp dashboard as well as make various alerts available. 

Connect to the website and **Login** as the master user.

Click **Home**

Click on the relevant **Chain**

Click on the relevant **Node**

Click the **Bell Icon** in the top right hand corner

On the left side Metrics are divided into **Metric Modules** of related values. Metrics are different depending on whether a Nodeos, Hyperion or Atomic Node is configured.

Various metrics are selectable within each **Metric Module** and are enabled by selecting **Monitor / Monitoring**

Once a specific metric is selected and **Saved**, it will be viewable in the dashboard and may open up options in the alert menu.


## 7) Set Alerts

Metrics only advise on a value while alerts will send a message when a certain condition is met. Depending on your configuration these alerts notify via the website dashboard, email, telegram and/or slack.

Alerts are set on each node and are available when a corresponding metric has been selected.

Connect to the website and **Login** as the master user.

Click **Home**

Click on the relevant **Chain**

Click on the relevant **Node**

Click the **Bell Icon** in the top right hand corner

Alerts are displayed on the right hand side and are grouped into **Alert Modules** of related values and typically would already have some default alerts applied. 

Each alert has a condition field which is what it will test against, there is an explanation of the condition field next to each alert.

### New User Alerts Recommendation
Start with experiment setting thresholds for alerts relevent to the Node Type, some Metrics will automatically be alerted on when selected such as Node/Connects or Node/Claimer_Has_Run

These alerts are quite handy to start with:
- Node/Db_Size_Free : 1024
- Node/HeadBlock : 100
- Node/Latency : 2000

### chains.json and bp.json alerts
Click **JSON Status**

Select the **Alerting** switch on the top right to alert on chains.json and bp.json status 

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
```

This will delete all container images with headsup-website and headsup-ingestor in them, except for the running containers. So as long as docker compose is running these will clean up old container images. The container images are qute large and useless once upgraded so you probably want to do this as part of your usual maintenace after performing an upgrade.

