# Snippets Text Subscription
This snippet will show you how to create a phrase based subscription service using LaML
## About Text Subscription
Easily create and maintain multiple campaigns, and their associated subscribers. The list administrator can broadcast to specific campaigns. The list administrator is notified of new subscribers via email, and of removal requests. If you reply with stop, or unsubscribe, the number will be placed in a black list.
## Getting Started
You will need a machine with Python installed, the SignalWire SDK, a provisioned SignalWire phone number, and optionaly Docker if you decide to run it in a container.

For this demo we will be using Python, but more languages may become available.

- [x] Python
- [x] SignalWire SDK
- [x] SignalWire Phone Number
- [x] MailGun Account (For administrator notifications, https://www.mailgun.com/)
- [x] Docker (Optional)
----
## Running Text Subscription - How It Works
## Methods and Endpoints

```
Endpoint: /text_inbound
Methods: GET OR POST
Endpoint to accept incoming text messages from SignalWire space.
Handles campaign subscriptions and removal requests
```

```
Endpoint: /broadcast_msg
Methods: GET OR POST
Send message to campaign subscribers by phrase and message.
```

## Setup Your Enviroment File

1. Copy from example.env and fill in your values
2. Save new file callled .env

Your file should look something like this
```
## This is the full name of your SignalWire Space. e.g.: example.signalwire.com
SIGNALWIRE_SPACE=
# Your Project ID - you can find it on the `API` page in your Dashboard.
SIGNALWIRE_PROJECT=
# Your API token - you can generate one on the `API` page in your Dashboard
SIGNALWIRE_TOKEN=
# The phone number you'll be using for this Snippets. Must include the `+1` , e$
SIGNALWIRE_NUMBER=
# MailGun domain associated with your MailGun account
MAILGUN_DOMAIN=
# MailGun token associated with your MailGun Account
MAILGUN_API_TOKEN=
# Send Email From Address
EMAIL_FROM=info@yourdomain.com
# Send email to address for administrator notifications
EMAIL_TO=youremail@yourdomain.com
# Email subject for admin notifications
EMAIL_SUBJECT=Text Subscription Admin Notification

```

## Setup Your Campaign File

1. Edit campaign.json to suit your needs, the file is a json object, that can handle multiple campaigns.

Your file should look something like this
```yaml
[
  {
    "Id": "1",
    "Name": "signalwire-demo-1",
    "Phrases": [
      "signalwire"
    ],
    "Subscribers": [
    ]
  },
  {
    "Id": "2",
    "Name": "signalwire-demo-2",
    "Phrases": [
      "original"
    ],
    "Subscribers": [
    ]
  }
]
```
## Setup Your Do Not Contact File

1. Edit donotcontact.json to suit your needs, the file is a json object and is global. This has no entries by default.

Your file should look something like this
```yaml
{
"DoNotContact": []
}
```

## Build and Run on Docker
Lets get started!
1. Use our pre-built image from Docker Hub 
```
For Python:
docker pull signalwire/snippets-text-subscription:python
```
(or build your own image)

1. Build your image
```
docker build -t snippets-text-subscription .
```
2. Run your image
```
docker run --publish 5000:5000 --env-file .env snippets-text-subscription
```
3. The application will run on port 5000

## Build and Run Natively
For Python
```
1. Replace environment variables
2. From command line run, python3 app.py
```

----
# More Documentation
You can find more documentation on LaML, Relay, and all Signalwire APIs at:
- [SignalWire Python SDK](https://github.com/signalwire/signalwire-python)
- [SignalWire API Docs](https://docs.signalwire.com)
- [SignalWire Github](https://gituhb.com/signalwire)
- [Docker - Getting Started](https://docs.docker.com/get-started/)
- [Python - Gettings Started](https://docs.python.org/3/using/index.html)
- [MailGun](https://www.mailgun.com/)

# Support
If you have any issues or want to engage further about this Signal, please [open an issue on this repo](../../issues) or join our fantastic [Slack community](https://signalwire.community) and chat with others in the SignalWire community!

If you need assistance or support with your SignalWire services please file a support ticket from your Dashboard. 
