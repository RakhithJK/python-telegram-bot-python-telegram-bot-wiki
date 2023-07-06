## Introduction
When your bot becomes popular, you will eventually want to improve response times. After all, Telegram places high priority on fast messaging. At the same time, responses become slower as more people are using your bot. This happens more quickly for inline bots, as they may receive multiple inline queries during one interaction. 

There are of course many ways to tackle this problem and not all work for everyone.
Before we list a few of them, here are some very important notes:

### Avoid premature optimization

Putting hours upon hours into squeezing every last bit of performance out of your code is worthless if your bot doesn't actually get much traffic.
Before you start optimizing, you should assess the actual usage of your bot.
For example, you can track the number of updates your bot receives along with a timestamp such that you can get an overview of the number of updates it processes on average.
Also check if your bot gets new updates in peaks or if they are spread evenly across the day.

### Avoid misdirected optimizations

Not all ways of improving performance will work for everyone.
More precisely, which methods to choose depend on which parts of your code slow down the bot the most.

If you do a lot of requests to the Bot API/an external API - i.e. I/O tasks -, concurrency may help.
OTOH, if you do a lot of heavy calculations, i.e. CPU-bound tasks, you can use `asyncio.create_task` all you want but you won't make much progress in terms of performance.

You should carefully analyze your bot and check e.g. which handler callbacks take the most runtime and check how to minimize that.

### Have realistic expectations

Keep in mind that you're programming in Python.
Immense speed is not something that Python is well known for.
If performance is more important to you than programming in Python, you might want to consider using a different programming language, e.g. a compiled language. 

## Free Optimizations

### PyPy
[PyPy](http://pypy.org/) is a different implementation of the Python programming language. Is your bot only using pure Python code? Using PyPy can then probably¹ optimize all your code -- free of charge.

¹PTB does *not* officially support PyPy as there have been a lot of issues in the past. It may still work out for you.

## Concurrency

Much of the work a usual bot does, boils down to making requests to the Bot API, i.e. network communication.
Such I/O operations usually involve a lot of *waiting*, i.e. time during which nothing really happens in your code.
Running I/O operations concurrently can hence save a lot of time.

This is probably one of the main motivations behind Pythons `asyncio` library, which PTB is built upon.
Because this topic is so important, it has its own [[wiki page|Concurrency]].

## Server location
Another potential bottleneck is the time your server (the computer that runs your bot script) needs to contact the Telegram server.
As of *June 2016*, there is only one server location for the Bot API, which is in the Netherlands.

**Note:** As of Bot API 5.0 you can also host your very own [[Bot API Server|Local-Bot-API-Server]].

### Test your connection

#### Using the ping utility
You can test your connection by running `ping api.telegram.org` on the command line of your server. A good connection should have a stable ping of 50ms or less. A server in Central Europe (France, Germany) can easily archive under 15ms, a server in the Netherlands reportedly archived 2ms ping. Servers in the US, Southeast Asia or China are not recommended to host Telegram bots.

#### Using the cURL utility
While the `ping` utility is helpful and the information is valuable, it's a rather primitive way to test your connection. In reality, there are many factors that influence the response times of your bot. For a more detailed test that actually connects to the Telegram servers via HTTPS, you can use cURL. The following is taken from [this blog post](https://josephscott.org/archives/2011/10/timing-details-with-curl/).

##### Step 1
On the server you want to test, create a file called `curl-format.txt` and paste this:

```
\n
            time_namelookup:  %{time_namelookup}\n
               time_connect:  %{time_connect}\n
            time_appconnect:  %{time_appconnect}\n
           time_pretransfer:  %{time_pretransfer}\n
              time_redirect:  %{time_redirect}\n
         time_starttransfer:  %{time_starttransfer}\n
                            ----------\n
                 time_total:  %{time_total}\n
\n
```

##### Step 2
Make a request to the Telegram API. In the following command, replace `<token>` with your API token and `<chat_id>` with your User ID (you can get your User ID from [@userinfobot](https://telegram.me/userinfobot)) and run it on your command line:

- Linux: `curl -w "@curl-format.txt" -o /dev/null -s "https://api.telegram.org/bot<token>/sendMessage?chat_id=<chat_id>&text=Test"`
- Windows: `curl -w "@curl-format.txt" -o NUL -s "https://api.telegram.org/bot<token>/sendMessage?chat_id=<chat_id>&text=Test"`

The result should look similar to this:

```
    time_namelookup:  0,004
       time_connect:  0,041
    time_appconnect:  0,119
   time_pretransfer:  0,119
      time_redirect:  0,000
 time_starttransfer:  0,156
                    ----------
         time_total:  0,156
```

**TODO:** Interpreting and improving these numbers.

**Note:** When choosing a server for the sole purpose of hosting a Telegram bot, these (ping and cURL) are the only relevant timings. Even if you are the only user of the bot, there is no advantage in choosing a server close to *you.* 

If you need some suggestions on where to host your bot, read [[Where to host Telegram Bots|Where-to-host-Telegram-Bots]].

# What to read next?
Learn [[how to use webhooks|Webhooks]] to get every last bit of performance from your bot.