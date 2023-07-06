# The Builder Pattern in `telegram.ext`

In this library, there are roughly four important components that make up everything:

1. The `Updater` is responsible for fetching updates that Telegram sent to your bot
2. The `Bot` provides high-level access to the methods of the Bot API
3. The `BaseRequest` is responsible to handle the actual networking stuff, i.e. sending the requests to the Bot API
4. The `Application` binds everything together and is responsible for handling the updates fetched by the `Updater`.

In addition to those four, there are several other components, which are not as significant for the structure of a `python-telegram-bot` program.

All of those components have different parameters. Some of them are optional. Some are required. Some are mutually exclusive.
That's a lot to take in and when coding your bot and setting this all up by yourself would be tiresome.

That's why `python-telegram-bot` makes an effort to make the setup easy with reasonable defaults.
For example, after running

```python
from telegram.ext import Application
application = Application.builder().token('TOKEN').build()
```

you will automatically have

* the `Updater` available as `application.updater`
* the `Bot` available as `application.bot` or `application.updater.bot` (both are the same object)
* a `BaseRequest` object initialized and ready to be used by the `application.bot`
* several other components & sane default values set up. 

But what if you want to customize some arguments that `Application`, `Updater`, `Bot`, `BaseRequest` or other components accept? Do you have to build all those objects yourself and glue them together? No! (Well, you can, but you don't have to.)

This is where the [builder pattern](https://en.wikipedia.org/wiki/Builder_pattern) comes into play. The idea is roughly as follows: You went shopping and have all the ingredients for a nice stew, but you don't want to cook yourself. So you hand everything to a chef. The chef will tell you that some of your ingredients don't match and will discard them. Afterwards, he'll cook a nice stew for you and you never need to worry about how exactly that's done.

Let's get a bit more technical. First, we need the cook:

```python
from telegram.ext import Application
builder = Application.builder()
```

Now, we hand over the ingredients:

```python
builder.token(token)  # the bot token is the main ingredient
builder.context_types(context_types)  # In case you want to use custom context types for your `Application`
builder.read_timeout(read_timeout)  # In case you want to fine tune the networking backend
...
```

Finally, we have the chef cook the stew:

```python
application = builder.build()
```

All this can also be chained into a single line:

```python
from telegram.ext import Application
application = Application.builder().token(token).context_types(context_types).read_timeout(read_timeout).build()
```

And that's already it!

The docs of [`ApplicationBuilder`](https://python-telegram-bot.readthedocs.io/telegram.ext.applicationbuilder.html) have all the info about which "ingredients" it can handle, i.e. which methods it has. Each method will tell you

* how the parameters will be used (e.g. the token passed to `ApplicationBuilder.token` will be used for the `Bot` available as `Application.bot`)
* What happens if you don't call this method. For most things, PTB will use reasonable defaults.
