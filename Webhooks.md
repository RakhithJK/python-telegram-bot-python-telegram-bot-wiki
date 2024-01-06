## Introduction
Our [examples](https://docs.python-telegram-bot.org/examples.html) usually start the bot using [`Application.run_polling`](https://docs.python-telegram-bot.org/telegram.ext.application.html#telegram.ext.Application.run_polling). This method uses the [getUpdates](https://core.telegram.org/bots/api#getupdates) API method to receive new updates for your bot. This is fine for smaller to medium-sized bots and for testing, but if your bot receives a lot of traffic, it might slow down the response times. There might be other reasons for you to switch to a webhook-based method for update retrieval.

**First things first:** You should have a good reason to switch from polling to a webhook. Don't do it simply because it sounds cool.

## Polling vs. Webhook
The general difference between polling and a webhook is: 

- Polling (via `get_updates`) periodically connects to Telegram's servers to check for new updates
- A Webhook is a URL you transmit to Telegram once. Whenever a new update for your bot arrives, Telegram sends that update to the specified URL.

Let's explain this with a metaphor.
Imagine you want to check if you have mail.

With (long) polling, you have a post office box and need to check yourself if you have new mail.
So you rush to the post office and wait in front of your box all day to see if the postman puts something in there - you only go home for small pee breaks.

With a webhook, you have a mailbox right at your home and the postman delivers the mail right to that mailbox. However, that only works if the postman knows your address (the URL).

## Requirements
### A public IP address or domain
Usually this means you have to run your bot on a server, either a dedicated server or a VPS. Read [[Where to host Telegram Bots|Where-to-host-Telegram-Bots]] to find a list of options. 

Make sure you can connect to your server from the **public internet**, either by IP or domain name. If `ping` works, you're good to go.

### A SSL certificate
All communication with the Telegram servers must be encrypted with HTTPS using SSL. With polling, this is taken care of by the Telegram Servers, but if you want to receive updates via a Webhook, you have to take care of it. Telegram will not send you any updates if you don't.

There are two ways to do this: 

1. A verified certificate issued by a trusted certification authority (CA)
2. A self-signed certificate

If you don't already have a verified certificate, use a self-signed one. It's easier and there is no disadvantage to it.

#### Creating a self-signed certificate using OpenSSL
To create a self-signed SSL certificate using `openssl`, run the following command:
```
openssl req -newkey rsa:2048 -sha256 -noenc -keyout private.key -x509 -days 3650 -out cert.pem
```

The `openssl` utility will ask you for a few details. **Make sure you enter the correct FQDN!** If your server has a domain, enter the full domain name here (eg. `sub.example.com`). If your server only has an IP address, enter that instead. If you enter an invalid FQDN (Fully Qualified Domain Name), you won't receive any updates from Telegram but also won't see any errors!

## Choosing a server model
There actually is a third requirement: a HTTP server to listen for webhook connections. At this point, there are several things to consider, depending on your needs. 

### The integrated webhook server
The `python-telegram-bot` library ships a custom HTTP server that is tightly integrated in the `telegram.ext` module and can be started using [`Updater.start_webhook`](https://docs.python-telegram-bot.org/en/stable/telegram.ext.updater.html#telegram.ext.Updater.start_webhook)/[`Application.run_webhook`](https://docs.python-telegram-bot.org/en/stable/telegram.ext.application.html#telegram.ext.Application.run_webhook). This webserver also takes care of decrypting the HTTPS traffic. It is probably the easiest way to set up a webhook.

However, there is a limitation to this solution. Telegram currently only supports four ports for webhooks: *443, 80, 88* and *8443.* As a result, you can only run **a maximum of four bots** on one domain/IP address with the integrated server. 

If that's not a problem for you (yet), you can use the code below (or similar) to start your bot with a webhook. The `listen` address should either be `'0.0.0.0'` or, if you don't have permission for that, the public IP address of your server. The port can be one of `443`, `80`, `88` or `8443`. It is recommended to set a secret token in the `secret_token` parameter, so no one can send fake updates to your bot. `key` and `cert` should contain the path to the files you generated [earlier](#creating-a-self-signed-certificate-using-openssl). The `webhook_url` should be the actual URL of your webhook. Include the `https://` protocol in the beginning, use the domain or IP address you set as the FQDN of your certificate and the correct port and URL path.

```python
application.run_webhook(
    listen='0.0.0.0',
    port=8443,
    secret_token='ASecretTokenIHaveChangedByNow',
    key='private.key',
    cert='cert.pem',
    webhook_url='https://example.com:8443'
)
```

> Remember to observe [the limitations for the secret token](https://docs.python-telegram-bot.org/en/stable/telegram.bot.html#telegram.Bot.set_webhook.params.secret_token), otherwise Telegram will not accept it.
>
> It obviously not such a good idea to store the value of `secret_token` right in your code, so consider creating an environment variable for it.

### Reverse proxy + integrated webhook server
To overcome the port limitation, you can use a reverse proxy like *nginx* or *haproxy*.

In this model, a single server application listening on the public IP, the *reverse proxy*, accepts all webhook requests and forwards them to the correct instance of locally running *integrated webhook servers.* It also performs the *SSL termination*, meaning it decrypts the HTTPS connection, so the webhook servers receive the already decrypted traffic. These servers can run on *any* port, not just the four ports allowed by Telegram, because Telegram only connects to the reverse proxy directly. 

Depending on the reverse proxy application you (or your hosting provider) are using, the implementation will look a bit different. In the following, there are a few possible setups listed.

#### Heroku
On Heroku using webhook can be beneficial because it will automatically manage the downtime required.
The reverse proxy is set up for you and an environment is created. From this environment you will have to extract the port the bot is supposed to listen on. Heroku manages the SSL on the proxy side, so you don't have to provide the certificate yourself.

```python
import os

TOKEN = "TOKEN"
PORT = int(os.environ.get('PORT', '8443'))
# add handlers
application.run_webhook(
    listen="0.0.0.0",
    port=PORT,
    secret_token='ASecretTokenIHaveChangedByNow',
    webhook_url="https://<appname>.herokuapp.com/"
)
```

#### Using nginx with one domain/port for all bots
This is similar to the Heroku approach, just that you set up the reverse proxy yourself. All bots set their `url` to the same domain and port. To differentiate the bots, add a different `url_path`. The integrated server should usually be started on the `localhost` or `127.0.0.1` address, the port can be any port you choose.

**Note:** `example.com` could be replaced by an IP address if you have no domain associated to your server.

Example code to start the bot:
```python
application.run_webhook(
    listen='127.0.0.1',
    port=5000,
    url_path='1',
    secret_token='ASecretTokenIHaveChangedByNow',
    webhook_url='https://example.com/TOKEN1',
    cert='cert.pem'
)
```

When setting up nginx, it is usually a good idea to secure it with an SSL certificate right away (see for example [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04)). If you've done that, you must not pass your certificate in `cert` parameter (if you do, Telegram will return an error).

Example configuration for `nginx` (reduced to important parts) with two bots configured:
```
server {
    listen              443 ssl;
    server_name         example.com;
    ssl_certificate     cert.pem;
    ssl_certificate_key private.key;

    location /TOKEN1 {
        proxy_pass http://127.0.0.1:5000/1/;
    }

    location /TOKEN2 {
        proxy_pass http://127.0.0.1:5001/2/;
    }
}
```

> If you try and reproduce the [custom webhook example](https://docs.python-telegram-bot.org/en/stable/examples.customwebhookbot.html) while using nginx as a reverse proxy, make sure you remove trailing slashes from the addresses in `proxy_pass` directive. A trailing slash [may cause status code 307 being emitted by Starlette](https://github.com/encode/starlette/issues/869) that is used in the example.

#### Using haproxy with one subdomain per bot
In this approach, each bot is assigned their own *subdomain*. If your server has the domain *example.com*, you could have the subdomains *bot1.example.com*, *bot2.example.com* etc. You will need one certificate for each bot, with the FQDN set for their respective subdomain (or a wildcard certificate). The reverse proxy in this example is `haproxy`. The integrated server should usually be started on the `localhost` or `127.0.0.1` address, the port can be any port you choose.

**Note:** For this to work, you need a domain for your server.

Example code to start the bot:
```python
application.run_webhook(
    listen='127.0.0.1',
    port=5000,
    secret_token='ASecretTokenIHaveChangedByNow',
    webhook_url='https://bot1.example.com',
    cert='cert_bot1.pem')
)
```

Example configuration for `haproxy` (reduced to important parts) with two bots configured. Again: The FQDN of both certificates must match the value in `ssl_fc_sni`. Also, the `.pem` files are the `private.key` file and `cert.pem` files concatenated:
```
frontend  public-https
    bind        0.0.0.0:443 ssl crt cert_key_bot1.pem crt cert_key_bot2.pem
    option      httpclose

    use_backend bot1 if  { ssl_fc_sni bot1.example.com }
    use_backend bot2 if  { ssl_fc_sni bot2.example.com }

backend bot1
    mode            http
    option          redispatch
    server          bot1.example.com 127.0.0.1:5000 check inter 1000

backend bot2
    mode            http
    option          redispatch
    server          bot2.example.com 127.0.0.1:5001 check inter 1000
```

### Custom solution
You don't necessarily have to use the integrated webserver *at all*. If you choose to go this way, **you should not use the `Updater` class.** The `telegram.ext` module was designed with this option in mind, so you can still use the `Application` class to profit from the message filtering/sorting it provides. You will have to do some work by hand, though.

The general idea is outlined below and also explained in more detail in [this wiki section](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Frequently-requested-design-patterns#running-ptb-alongside-other-asyncio-frameworks).

```python
from telegram import Bot
from telegram.ext import Application

application = Application.builder().token('TOKEN').build()
    
# Register handlers here

# Get the update_queue from which the application fetches the updates to handle
update_queue = application.update_queue
start_fetching_updates(update_queue)

# Start and run the application
async with application:
    application.start()
    # when some shutdown mechanism is triggered:
    application.stop()
```

Here, `start_fetching_updates` is a placeholder for whichever method you use to set up a webhook.
The important part is that you enqueue the received updates into the `update_queue`.
That is, call `await update_queue.put(update)`, where `update` is the decoded `Update` object (use `Update.de_json(json.loads(text), bot)` to decode the update from the received JSON data).

We have an example using this approach available [here](https://docs.python-telegram-bot.org/examples.html#examples-customwebhookbot).

#### Alternative: No long running tasks
If you don't want to use the long running tasks started by `application.start()`, you don't have to!
Instead of putting the updates into the `update_queue`, you can directly process them via `application.process_update(update)`.

## Checking your webhook
To check if the webhook was accepted by Telegram, you can simply open this link in your browser:
`https://api.telegram.org/bot<your_bot_token>/getWebhookInfo`
