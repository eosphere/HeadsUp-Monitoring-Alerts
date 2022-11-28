# Architechure

A running HeadsUp implementation consists of 3 components Website, Ingestor and Database.

HeadsUp is designed for Antelope Node Operators to run a private instance of the platform on their internal network using docker for each of the components and docker compose to orchestrate and configure the base deployment.

## Website

The Website is the central hub of the HeadsUp platform. The website receives metrics and alerts from the Ingestor (over a websocket) and forwards them onto whoever needs to receive them. This can be the website frontend, email or slack recipients. 

The website is a SPA (single page application) when the user browses to it, it opens a websocket to the website. The website then sends metrics and alerts down that websocket.

## Ingestor
???

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







The best way to install Headsup is using docker-compose. The file `config/docker-compose/docker-compose.yaml` contains an example docker-compose file.

First install docker and docker-compose on your server.

Place the docker-compose.yaml file in a convenient place on the server. I just put it in the home directory of the user I was using to run it.

Edit the docker-compose.yaml on the server (I used nano)

In the ports entry of the website section the incoming port of the webserver is set to 8000. You will probably want to change this to the port you want to use. Setting up these ports and the firewalls is not covered in this tutorial and left up to you.

Set the following entries

**headsup_server_masterpassword** is the master password for the single user who has the power to configure the system. Keep it a secret to prevent others making malicious changes to your server. This is a web user login, Headsup does not maintain a list of user and password pairs there is only one user with one password.

**headsup_server_jwtsecret** is the random seed used to generate a private key to sign authentication tokens. The most secure way to generate this is to use a password generator. If the master password above leaks you should change this value in addition to the master password.

**headsup_server_debugingestorpassword** should not be set for production systems.

**headsup_server_emailalerts** Set to true if you want to send email alerts.

**headsup_server_emailsender** The from address for email alerts

**headsup_server_emailto** The address to send alerts to

**headsup_server_emailsmtpserver** The address of the SMTP server to send the emails to

**headsup_server_emailsmtpssl** Use ssl for the connection to the email server

**headsup_server_emailsmtptls** Use tls for the connection to the email server (I beleive this is actually StartTLS)

**headsup_server_emailsmtpport** Port to connect to the SMTP server on

**headsup_server_emailsmtpuser** User name to use when connecting to the SMTP server

**headsup_server_emailsmtppassword** Password to use when connecting to the SMTP server

**headsup_server_rooturl** The root of your webserver, used for calculating the full URLs for silencing (eg https://headsup.eosphere.io)

**headsup_server_slackalerts** Set to true if want to send slack alerts

**headsup_server_slackhook** The slack hook URL for your postgres

## First run

I have found the byobu program to be very useful for this part as it enables me to easily switch between different command line applications.

We want things to initialise in the correct order so we will start the containers in sequence.

First the postgres database container.

`docker-compose up postgres`

This will download the postgres container and then run it which initialises the database. We do this because of the time it takes to initialise this container. When the website container starts it tries to run the database migrations immediately, if the database is not ready this causes problems.

Once the database container is initialised press Ctrl+C to stop it.

Next we want to start the website.

`docker-compose up website`

This is take a while to download the container and start it. Once it has started up go to it in the web browser. Then login with the master password.

Click the Setup entry on the left hand side. 
Click the plus sign next to ingestors
Give your Ingestor a name (I used Ingestor 1) and set the sort order to 0.

Click save then click the Save button and once it has saved click the Ingestor Access Key button (picture of arrow and lock). Copy the code it displays to a notepad for safe keeping. Click the set button to update the server.

The access key is like the password for the Ingestor and the Name is like it's user name.

Edit the docker-config.yaml file to place the new access_key in the headsup_accesskey in the ingestor section.

Go back to the command line on the server where docker-compose is running press Ctrl+C and type in docker-compose up

This will download and start the ingestor container. Once it has done this you should see messages in the console saying `Ingestor Ping Enqueue` and `Sending ack for ingestor ping` this means the ingestor is running and talking to the website container.

Next stop docker-compose again by pressing Ctrl+C and running `docker-compose up -d` this will run docker-compose in the background. You can see the running docker processes with `docker ps`.

In this config docker-compose should automatically restart on reboot. Reboot the server and after the restart log in and check this with the `docker ps` command.

## Useful commands while running Headsup

### Viewing logs

`journalctl -u docker.service -n 25`

Show the last 25 log entries.

`journalctl -f -u docker.service`

Stream logs to the console as they occur. Useful for detemining the liveness of headsup

### Editing docker-compose.yaml

`byobu` - allows you to run multiple consoles at the same time. Similar to the old screen command. I find it useful to run the streaming logs in one byobu console while performing an upgrade.

### Cleaning up old containers

```
docker images | grep headsup-website | tr -s ' ' | cut -d ' ' -f 2 | xargs -I {} docker rmi mlockett42/headsup-website:{}
docker images | grep headsup-ingestor | tr -s ' ' | cut -d ' ' -f 2 | xargs -I {} docker rmi mlockett42/headsup-ingestor:{}
```

These delete all container images with headsup-website and headsup-ingestor in them, except for the running containers. So as long as docker compose is running they will clean up old container images. The container images are qute large and useless once upgraded so you probably want to do this regularly

## Configuring chains

Connect to the website and log in as the master user.

Click `Home`

You should see a blank screen as you have not set up any chains.

Click `Add Chain`

You have the following fields

**Chain Name**
A human readable name for the chain. This name is just for HeadsUp's user interface's benefit.

**Chain Type**
You can choose the chain type, this will prefillout some of the values (chain id and Vote Decay Type) with that chain's known settings and use the correct chain icon.

**Icon**
Icon to use for this chain, you cannot customise this setting.

**BP Name**
For metrics which monitor our block producer this is the name of the block producer. Ie the name of the owning EOS account.

**Vote Decay Type**
The original EOS mainnet halved the values of votes after 52 weeks, WAX uses 13 weeks and Telos doesn't have vote decay any more at all. You can only use one of the preset options

**Chain Id**
The EOSIO Chain Id. Each Mainnet and Testnet (and private network) will have one of these.

**Sort Order**
The order in which to display this chain in the on-screen list.

The chain will be added to the left hand panel you can then click `Add Node`

## Configuring Nodes

You have the following fields
**Node Name**
A human readable name for the node. This name is just for HeadsUp's user interface's benefit.

**Node Type**
Available types
Block producer - The lead Nodeos block producer
Nodeos follower node - The secondary Nodeos instance
Hyperion
Atomic Assets
Website - monitor and alert on the chains.json and bp.json files

The difference between "Block Producer" and "Nodeos Follower" in the head block number metric. For the purposes of the alert the block number of the follower node in compared to the block number of the Block Producer node.

**IP Address**

Address of the node, can also be a DNS name

**Port**
Port to connect to the node on

**Sort Order**
On screen display order of the nodes instead Headsup

**Use SSL for connections**
Connect via HTTPS otherwise unencrypted HTTP

**Ingestor**
It is possible to have multiple ingestors here we choose which one will connect to and test the node.

## Configuring Metrics

Metrics are things we measure and display in the web app. To set them go to the node and then click the bell icon in the top right hand corner.

Metrics are devided into modules of related values, eg all value derived directly from a Nodeos node are grouped together. All from a Hyperion node together etc. One odd one is Voting related.

### Available Metrics

**Node/Latency**

Time in milliseconds to connect to the node

**Node/Connects**

Do you connect at all to the node

**Node/Head_Block**

The head block number

**Node/Chain_Id**

Chain Id the node reports

**Node/Last_Irreversible_Block**

The node reported last irreversible block

**Node/Server_Version**

The nodes report software version

**Node/Head_Block_Producer**

The node reported current head block producer

**Node/Supported_Apis_Advertised**

A list of the support apis

**Node/Db_Size**

Reported size of the database

**Node/Claimer_Has_Run**

Have BP rewards been claimed in the last 24 hours

**Vote/Total**

Total number of votes for our BP

**Vote/Rank**

Our block producer's position in the vote rankings

**Vote/Percentage**

Our votes as a percentage

**Vote/ActiveProducerHasProduced**

If our producer is in the top 21 has it produced when it should have. This will alert if the block producer is in the top 21 but hasn't produced

**Website/ChainsJsonAndBpJson**

Are the chains.json and bp.json files set up correctly.

**History_Api/Connects**

Is the legacy History API accepting connections

**Hyperion/Latency**

Time in milliseconds to connect to the node

**Hyperion/Connects**

Can we connect to the node

**Hyperion/Version**

Version of the hyperion software

**Hyperion/Features**

List of available feature reported by the HyperionS oftware

**Hyperion/NodeosRpc**

Is the NodeosRpc service reported in good health by Hyperion

**Hyperion/ElasticSearchStatus**

Is the Elastic Search service reported in good health by Hyperion

**Hyperion/ElasticSearchIndexedBlock**

What is the last reported block indexed by elastic search

**Hyperion/BlockNumber**

What is the last Nodeos block reported to Hyperion

**AtomicAssets/Latency**

Time in milliseconds to connect to the node

**AtomicAssets/Connects**

Can we connect to the node

**AtomicAssets/Version**

Software version as reported by the node

**AtomicAssets/Status**

Status as reported by the node

**AtomicAssets/BlockNumber**

The last Eosio block number processed by the node

### Setting Alerts

Metrics only tell us a value while alerts will send us a message when a certain condition is met. There is a alerts tab in the same dialog where you enable metrics. Switch to it and you can enable alerts on that metric. Each alert has a condition field which is what it will test against and an explanation of what the condition field is.


