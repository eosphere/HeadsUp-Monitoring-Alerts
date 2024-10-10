This guide will cover how to configure and start receiving HeadsUp Alerts through Telegram using the Bot API. The procedure may seem a bit tricky but it’s actually really simple and should only take a few minutes.

# Configure HeadsUp Telegram Alerts

Using the Telegram Bot API and configuring the HeadsUp  `docker-compose.yaml`  the steps to receiving HeadsUp Alerts via Telegram are the following:

1.  Create and Start a new Telegram Bot
2.  Create a new Alert Group and add your Telegram Bot
3.  Obtain the Group Chat ID
4.  Configure and Restart HeadsUp

# Create and Start a new Telegram Bot

Begin by connecting to BotFather the Telegram Bot API, either though a browser  [https://t.me/BotFather](https://t.me/BotFather)  or searching for  `BotFather`  in Telegram.

![image](https://github.com/user-attachments/assets/ab28b543-03f7-4d93-a21b-3d2d42a6f12a)

Then create a new bot, by choosing a name and username. Take note of the Token used to access the HTTP API, save it and keep it safe.

![image](https://github.com/user-attachments/assets/4b1e301f-40f1-4891-a358-715743d6bed7)

In Telegram go to your newly created Bot and click  **START**

![image](https://github.com/user-attachments/assets/388c4168-b0aa-4fda-8387-57283abac3a5)

![image](https://github.com/user-attachments/assets/c8f5587a-50c9-4569-ae4b-cf8d36f9b215)

# Create a new Alert Group and add your Telegram Bot

Assign a name to your new group

![image](https://github.com/user-attachments/assets/2abf0445-f2d5-40cf-bde3-f27bb14cce06)

Add your Telegram Bot

![image](https://github.com/user-attachments/assets/31174ee6-57d6-42ef-92ba-52a04e4d4bca)

This will be the group from where your Guild will receive all alerts and notifications from HeadsUp. Add your Guild member Telegram accounts to this group as required.

# Obtain the Group Chat ID

From within your Telegram Group run this message using your Bot username to make the  **Group Chat ID** visible.

_This is not always required, most times the Group Chat ID is instantly visible when the Bot is added to the group during creation._

`/my_id@headsup_alerts_demo_bot`

![image](https://github.com/user-attachments/assets/c74cb410-347b-4ae8-b94a-0ad0cd5898a4)

Then from a browser query the Telegram API with your saved Telegram HTTP API Token in the following format:

`https://api.telegram.org/bot<TOKEN HERE>/getUpdates`

In this example it will look like this:

`https://api.telegram.org/bot7017987013:AAG3EzGxBn7-e_PR_nWUWx7I2EFvkvyvmDo/getUpdates`

Be sure not to forget the  `/bot`  and  `/getUpdates`  and each end of your token.

The output will have your Group Chat ID displayed as below, take note of the  **Group Chat ID**:

```
{  
      "update_id": 759577033,  
      "message": {  
        "message_id": 8,  
        "from": {  
          "id": 251151145,  
          "is_bot": false,  
          "first_name": "Ross",  
          "last_name": "(EOSphere)🇦🇺",  
          "username": "rossco99",  
          "language_code": "en"  
        },  
        "chat": {  
          "id": -4546176026,  <-- HERE
          "title": "Demo HeadsUp Alerts",  
          "type": "group",  
          "all_members_are_administrators": true  
        },  
        "date": 1727927284,  
        "text": "/my_id@headsup_alerts_demo_bot",  
        "entities": [  
          {  
            "offset": 0,  
            "length": 30,  
            "type": "bot_command"  
          }
```

# Configure and Restart HeadsUp

Now that we have the  **HTTP API Token**  and  **Group Chat ID,** HeadsUp may be configured with your specific details as below:

```
nano docker-compose.yaml  
  
website:  
  environment:  
      - headsup_server_telegramalerts=True                                                                                                                                                                                              
      - headsup_server_telegrambot=7017987013:AAG3EzGxBn7-e_PR_nWUWx7I2EFvkvyvmDo                                                                                                                                                       
      - headsup_server_telegramchat=-4546176026
```

Restart HeadsUp:

```
docker compose up -d
```

You should now be receiving Alerts directly into your Telegram Group, ensure you add your relevant Guild members and test that all is working as expected.

![image](https://github.com/user-attachments/assets/0036b28e-e0ec-4ab2-b5c5-fa237495e2e3)
