A `Handler` is an instance derived from the base class [`telegram.ext.Handler`](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.handler.html#telegram.ext.Handler) which is responsible for the routing of different kinds of updates (text, audio, inlinequery, button presses, ...) to their _corresponding callback function_ in your code.

For example, if you want your bot to respond to the command `/start`, you can use a [`telegram.ext.CommandHandler`](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.commandhandler.html) that maps this user input to a callback named `start_callback`:
```python
def start_callback(update, context):
    update.message.reply_text("Welcome to my awesome bot!")

...

dispatcher.add_handler(CommandHandler("start", start_callback))
```

## Different types of `Update`s

For different kinds of user input, the received `telegram.Update` will have different attributes set. For example an incoming message will result in `update.message` containing the sent message. The pressing an inline button will result in `update.callback_query` being set. To differentiate between all those updates, `telegram.ext` provides

1) [`telegram.ext.MessageHandler`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.messagehandler.html) for all message updates
2) multiple handlers for all the other different types of updates, e.g. [`telegram.ext.CallbackQueryhandler`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.callbackqueryhandler.html) for `update.callback_query` and [`telegram.ext.InlineQueryHandler`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.inlinequeryhandler.html) for `update.inline_query`

The special thing about `MessageHandler` is that there is such a vast variety of message types (text, gif, image, document, sticker, …) that it's infeasible to provide a different `Handler` for each type. Instead `MessageHandler` is coupled with so called [filters](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.filters.html) that allow to make fine-grained distinctions: `MessageHandler(Filters.all, callback)` will handle all updates that contain

* `update.message`
* `update.edited_message`
* `update.channel_post`
* `update.edited_channel_post`

You can use the different filters to narrow down which updates your `MessageHandler` will handle. See also [this article](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Extensions-–-Advanced-Filters) for advanced usage of filters.

## CommandHandlers with arguments

It is also possible to work with parameters for commands offered by your bot. Let's extend the `start_callback` with some arguments so that the user can provide additional information in the same step:

```python
def start_callback(update, context):
    user_says = " ".join(context.args)
    update.message.reply_text("You said: " + user_says)

...

dispatcher.add_handler(CommandHandler("start", start_callback))
```

Sending "/start Hello World!" to your bot will now split everything after /start separated by the space character into a list of words and pass it on to the `args` parameter of `context`: `["Hello", "World!"]`. We join these chunks together by calling `" ".join(context.args)` and echo the resulting string back to the user.

### Deep-Linking start parameters
The argument passing described above works exactly the same when the user clicks on a deeply linked start URL, like this one:

[https://t.me/roolsbot?start=Hello_World!](https://t.me/roolsbot?start=Hello_World!)

Clicking this link will open your Telegram Client and show a big START button. When it is pressed, the URL parameters "Hello_World!" will be passed on to the `args` of your context object.

Note that since telegram doesn't support spaces in deep linking parameters, you will have to manually split the single `Hello_World` argument, into `["Hello", "World!"]` (using `context.args[0].split('_')` for example)

You also have to pay attention to the maximum length accepted by Telegram itself. As stated in the [documentation](https://core.telegram.org/bots#deep-linking) the maximum length for the start parameter is 64.

Also, since this is an URL parameter, you have to pay attention on how to correctly pass the values in order to avoid passing URL reserved characters. Consider the usage of `base64.urlsafe_b64encode`.

## Pattern matching: `Filters.regex`

For more complex inputs you can employ the [`telegram.ext.MessageHandler`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.messagehandler.html) with [`telegram.ext.Filters.regex`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.filters.html#telegram.ext.filters.Filters.regex), which internally uses the `re`-module to match textual user input with a supplied pattern.

Keep in mind that for extracting URLs, #Hashtags, @Mentions, and other Telegram entities, there's no need to parse them with a regex filter because the Bot API already sends them to us with every update. Refer to [this snippet](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Code-snippets#message-entities) to learn how to work with entities instead.

This tutorial only covers some of the available handlers (for now). Refer to the documentation for all other types: https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.html#handlers

## Custom updates

In some cases, it's useful to handle updates that are not from Telegram. E.g. you might want to handle notifications from a 3rd party service and forward them to your users. For such use cases, PTB provides

* [`TypeHandler`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.typehandler.html)
* [`StringCommandHandler`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.stringcommandhandler.html)
* [`StringRegexHandler`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.stringregexhandler.html)
