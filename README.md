# Text Subscription Service

This guide will show you how to create a phrase-based subscription service using the [Python SignalWire SDK](https://developer.signalwire.com/twiml/reference/client-libraries-and-sdks#python). Easily create and maintain multiple campaigns and their associated subscribers. The list administrator can broadcast to specific campaigns. The list administrator is notified of new subscribers via email, and of removal requests. If you reply with stop or unsubscribe, the number will be placed in a black list.

# Setup Your Environment File

1. Copy from example.env and fill in your values
2. Save new file called .env

Your file should look something like this.
```
## This is the full name of your SignalWire Space. e.g.: example.signalwire.com
SIGNALWIRE_SPACE=
# Your Project ID - you can find it on the `API` page in your Dashboard.
SIGNALWIRE_PROJECT=
# Your API token - you can generate one on the `API` page in your Dashboard
SIGNALWIRE_TOKEN=
# The phone number you'll be using for this guide. Must include the `+1`, 
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

# Setup Your Campaign File

1. Edit campaign.json to suit your needs, the file is a JSON object that can handle multiple campaigns.

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
# Setup Your Do Not Contact File

1. Edit donotcontact.json to suit your needs, the file is a JSON object and is global. This has no entries by default.

Your file should look something like this
```yaml
{
"DoNotContact": []
}
```

# Configuring the Code
The first step is to define a function `send_email(body)` that will utilize the MailGun API to send an email using the variables from our `.env` file.

```python
# MailGun Send Email
def send_email(body):
    # Post to MailGun to shoot out an email
    return requests.post(
        "https://api.mailgun.net/v3/" + os.environ['MAILGUN_DOMAIN'] + "/messages",
        auth=("api", os.environ['MAILGUN_API_TOKEN']),
        data={"from": os.environ['EMAIL_FROM'],
              "to": [os.environ['EMAIL_TO']],
              "subject": os.environ['EMAIL_SUBJECT'],
              "text": body })
```

Next, we will define a quick function to help us turn a JSON object into a string. 

```python
# JSON serialization helper
def set_default(obj):
    if isinstance(obj, set):
        return list(obj)
    raise TypeError
```

Now that we have defined the two necessary functions that we will call in this process, we can move on to our routes that will handle GET/POST requests! The first route we will use is `/text_inbound` which will accept GET/POST requests for inbound text messages. 

The first part of this route is for handling **opt-out messaging**. We will open both the `campaigns.json` file and the `donotcontact.json` file, and get the `body` and from `number` parameters from the HTTP request. We will check to make sure that the `number` is not on the do not contact list - if it is, we will ignore the request. Then we will check if the message `body` includes any version of "unsubscribe" or "remove" or "stop". If it does, we will add them to the do not contact list and send them an unsubscribed message. We will then use the `send_email(body)` function to send an email to an administrator that the subscriber has opted out of further messaging. 

The next part is how to handle **opt-in messaging**. Here we will loop through all of the campaigns in the `campaign.json` file. If the number is already subscribed, we will send them a message to let them know. If not, we will add them to our `campaign.json` file and send them a message to thank them for subscribing. 

```python

# Listen on route '/text_inbound' for inbound text messages on GET/POST requests
@app.route('/text_inbound', methods=['GET', 'POST'])
def text_inbound():

    # Initialize the SignalWire client
    client = signalwire_client(os.environ['SIGNALWIRE_PROJECT'], os.environ['SIGNALWIRE_TOKEN'], signalwire_space_url = os.environ['SIGNALWIRE_SPACE'])

    # Read campaigns from json file
    with open('campaigns.json') as f:
        campaigns = json.load(f)

    # Read Do Not Contact Json
    with open('donotcontact.json') as f:
        donotcontact = json.load(f)

    # Read params passed in by request
    phrase = request.values.get("Body")
    number = request.values.get("From")

    print(donotcontact)
    print(number)

    # If number is on do not contact list, ignore request
    for x in donotcontact['DoNotContact']:
        print(x[0])
        if number == x[0]:
            return "200"

    # Trim the phrase provided
    phrase = phrase.strip()

    # Check phrase for STOP / UNSUBSCRIBE
    if phrase.lower() == "stop" or phrase.lower() == "unsubscribe" or phrase.lower() == "remove":

        # Add number to DoNotContact file
        donotcontact['DoNotContact'].append( { number } )

        # Write updated DoNotContact to file
        with open('donotcontact.json', 'w') as f:
            print(donotcontact)
            json.dump(donotcontact, f, default=set_default)

        # Send receipt of unsubscribe message
        message = client.messages.create(
            from_ = os.environ['SIGNALWIRE_NUMBER'],
            body = "You have been removed from our list.",
            to = number
        )

        # Send email to administrator or your hook logic, unsubscribed
        send_email("Subscriber requested to be removed: \n" + " Number: " + number)

        return "200"

    # Loop through all campaigns for active phrase
    for campaign in campaigns:

        if phrase in campaign["Phrases"]:

            # If the number is in a campaign then they are already subscribed, else add them to the campaign
            for x in campaign['Subscribers']:

                if number == x[0]:

                    # Send already subscribed message
                    message = client.messages.create(
                        from_ = os.environ['SIGNALWIRE_NUMBER'],
                        body = "You are already subscribed to '" + phrase  + "'",
                        to = number
                    )

                else:

                    # Add new number to campaign subscriber list
                    campaign["Subscribers"].append( { number } )

                    # write updated data to file
                    with open('campaigns.json', 'w') as f:
                        json.dump(campaigns, f, default=set_default)

                    # Send message
                    message = client.messages.create(
                        from_ = os.environ['SIGNALWIRE_NUMBER'],
                        body = "Thank you for subscribing to '" + phrase  + "'",
                        to = number
                    )

                    # Send email to administrator or your hook logic
                    send_email("New subscriber to campaign: " + phrase + "\n" + " Number: " + number)

    return "200"
```
The next endpoint `/broadcast_msg` can be called by the list administrator in order to send out messages. When this endpoint is called along with the `code` and `message` parameters, it will open the `campaign.json` file, loop through the subscribers, and send out the messages. 


```python
# Listen for route/requests at endpoint
@app.route('/broadcast_msg', methods=['GET','POST'])
def broadcast_msg():

    # Initialize the SignalWire client
    client = signalwire_client(os.environ['SIGNALWIRE_PROJECT'], os.environ['SIGNALWIRE_TOKEN'], signalwire_space_url = os.environ['SIGNALWIRE_SPACE'])

    # Read phrase param, that represents campaign subscribers
    group_code = request.values.get("code")
    # Read message param, that represents message to be sent 
    message = request.values.get("message")

    # Read campaigns from json file
    with open('campaigns.json') as f:
        campaigns = json.load(f)

    # Loop through subscribers, and send messages
    for number in campaigns['Subscribers']:
        message = client.messages.create(
            from_ = os.environ['SIGNALWIRE_NUMBER'],
            body = message,
            to = number
        )

    return "200"

```

# Build and Run on Docker

1. Use our pre-built image from Docker Hub 
```
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

# Build and Run Natively

To run the application, execute export FLASK_APP=app.py then run flask run.

You may need to use an SSH tunnel for testing this code if running on your local machine. – we recommend [ngrok](https://ngrok.com/). You can learn more about how to use ngrok [here](https://developer.signalwire.com/apis/docs/how-to-test-webhooks-with-ngrok). 

# Sign Up Here

If you would like to test this example out, you can create a SignalWire account and space [here](https://m.signalwire.com/signups/new?s=1).

Please feel free to reach out to us on our Community Slack or create a Support ticket if you need guidance!
