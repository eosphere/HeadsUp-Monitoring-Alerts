This guide will cover how to configure and receive HeadsUp Alerts through Slack using an Incoming Webhook App. The procedure is quite simple especially if you are already using a Slack Workspace.

# Configure HeadsUp Slack Alerts

Using a Slack Incoming Webhook and configuring the HeadsUp  `docker-compose.yaml`  the steps to receiving HeadsUp Alerts via Slack are the following:

1.  Create a Slack Workspace (Or use an existing one)
2.  Create a New Slack Channel for HeadsUp Alerts
3.  Create a New Slack App
4.  Activate Incoming Webhooks
5.  Configure and Restart HeadsUp

# Create a Slack Workspace

If you aren't already using Slack and don’t have an existing Workspace,  [sign up to Slack](https://slack.com/)  with your preferred email by selecting  **GET STARTED.**

![image](https://github.com/user-attachments/assets/6849cfbe-3871-42ff-9004-85a46ae87b88)

Then create a new workspace for use by your Team and HeadsUp by selecting  **Create a workspace**.

![image](https://github.com/user-attachments/assets/32ee6025-771c-4227-96cb-935cfa2376ba)

Follow the  **Create Workspace Wizard**  to create a workspace name, create your identity, add team members and choose your subscription model (Free gives you 90 days of message history).

In this example we have create a Workspace called  **HEADSUP TEST**

![image](https://github.com/user-attachments/assets/17d9c13c-6e14-402a-afe6-bd0a06c85f88)

Select  **LAUNCH SLACK**  to open the workspace in browser or use the Slack Desktop App.

![image](https://github.com/user-attachments/assets/9226a594-3e75-4db2-be77-b316cf1fef7f)

# Create a New Slack Channel for HeadsUp Alerts

Within this workspace we can now create a specific channel for receiving HeadsUp Alerts.

![image](https://github.com/user-attachments/assets/3a167cc3-9ed3-49b5-bfe4-ab9ea88ab60b)

![image](https://github.com/user-attachments/assets/674a7833-b8bb-4589-84be-5479092107c2)

In this example we have create a channel for receiving alerts called  **#headsup**

![image](https://github.com/user-attachments/assets/87b45b99-630e-4c94-924a-1bb1491114fc)

# Create a New Slack App

Navigate to  **Workspace settings**

![image](https://github.com/user-attachments/assets/8ae2c057-c230-4a00-aab6-9ad1bcfa4080)

Select  **Configure Apps**

![image](https://github.com/user-attachments/assets/f13ce00c-78df-48de-8402-462d1ffddf94)

Select  **Build**  in the top right

![image](https://github.com/user-attachments/assets/adadc0d8-82f3-4972-a04a-cd9243da193d)

Select  **Create New App -> From scratch**,  provide an  **App Name**  and  **Pick a workspace**, then select  **Create App.**

![image](https://github.com/user-attachments/assets/8c04c201-d7f2-450d-9a79-800750339ec2)

Once created you are now able to customise the App including providing an Icon and Description.

# Activate Incoming Webhooks

Next we need to Activate Incoming Webhooks and generate a Webhook URL that can be used by HeadsUp for alerting.

Select  **Incoming Webhooks** in the **Features**  section

![image](https://github.com/user-attachments/assets/f6ce2d56-4a22-4e07-ba73-b062b7095e81)

Turn the selector to  **On**

![image](https://github.com/user-attachments/assets/4235783a-7df3-4665-a148-d6c7ec3b1e39)

Then select  **Add New Webhook to Workspace**

![image](https://github.com/user-attachments/assets/4391a59b-716b-48d6-869b-fa12aaa64aa6)

Select the channel to be used for this Webhook and  **Allow**  access.

![image](https://github.com/user-attachments/assets/3dde1035-1a9a-40f1-a920-5b85c589af57)

The Slack Webhook URL is now available, copy this URL so we can use it in the HeadsUp configuration.

![image](https://github.com/user-attachments/assets/a2111150-7e9c-4677-9394-22348948049b)

For simplicity you can always directly access the  [Slack API Apps](https://api.slack.com/apps)  page directly through this URL -> ```https://api.slack.com/apps```

# Configure and Restart HeadsUp

Now that we have the  **Slack Webhook URL,** HeadsUp may be configured with your specific details as below:

```
nano docker-compose.yaml  
  
website:  
  environment:  
      - headsup_server_slackalerts=True  
      - headsup_server_slackhook=https://hooks.slack.com/services/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
```

Restart HeadsUp:

```
docker compose up -d
```

You should now be receiving Alerts directly into your Slack Workspace specific Channel, ensure you add your relevant Team members to this channel and test that all is working as expected.

![image](https://github.com/user-attachments/assets/64ddf75b-5f03-43c5-bb8c-78b0826552c0)
