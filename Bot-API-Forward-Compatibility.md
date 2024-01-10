The `telegram` package contains PTBs Python implementation of the classes and methods defined in the [Bot API](https://core.telegram.org/bots/api).
This implementation is manually updated whenever Telegram releases a new version of the API.
Due to the nature of this open-source library, the implementation work usually takes some time, dependin on how much changed.
However, users of PTB might want to immediately use the new functionality provided by Telegram.
PTB therefore provides mechanism that make exactly this possible, i.e. accesing information from the Bot API that has no native PTB implementation yet.
Below, we will describe the mechanisms in detail.

> [!Warning]
> It is important to note that all these mechanism are intended to be used as temporary solutions only. 
> If you use these mechanisms in your code and then upgrade to a PTB version that adds native support for the new Bot API functionality, you'll have to adapt your code to use PTBs native interfaces instead.

# Object Attributes

## Receiving Objects

Telegram sends data in the form of JSON objects either for updates that your bot receives or as return value of methods.
When Telegram adds new attributes ("fields") to these objects that, the corresponding attributes in PTBs classes will be missing.
However, any additional data that is not yet covered by the native Python attributes will be gathered in the [`TelegramObject.api_kwargs`](https://docs.python-telegram-bot.org/en/stable/telegram.telegramobject.html#telegram.TelegramObject.api_kwargs) attribute (note that `TelegramObject` is the base class of (almost) all other classes in the `telegram` package).

For example, imagine that Telegram adds a new field called `read_date` to the `Message` class. Then accessing `update.message.read_date` is possible only after PTB was updated, however accessing `update.message.api_kwargs["read_date"]` would give the value provided by Telegram.

> [!Important]
> `TelegramObject.api_kwargs` will always contain the raw JSON data received from Telegram. For example, a date will be a simple unix timestamp instead of a timezone aware `datetime.datetime` object. These kinds of conversions are available only once PTB is updated to the new API version.

As another example, imagine that Telegram adds a new filed called `user_online_status_updated` to the `Update` class. Then accessing `update.user_online_status_updated` is again not immediately possible. Moreover, there is not yet a handler in `telegram.ext` to handle these new kinds of updates. However, in this case you can e.g. use [`TypeHandler`](https://docs.python-telegram-bot.org/en/stable/telegram.ext.typehandler.html) to catch these updates, like this

```python
from telegram import Update
from telegram.ext import TypeHandler

async def callback(update, context):
    if status_updated := update.api_kwargs.get("user_online_status_updated"):
        print(status_updated)

application.add_handler(TypeHandler(Update, callback))
```

Of course you can also ipmlement your own subclass of [`BaseHandler`](https://docs.python-telegram-bot.org/en/stable/telegram.ext.basehandler.html) that checks for the presence of the key `user_online_status_updated` in `update.api_kwargs`. See also [[this page|Types-of-Handlers]].

## Sending Objects

The Bot API also receives objects from you, namely as input arguments for the different Bot methods. For example, [`Bot.set_my_command`](https://docs.python-telegram-bot.org/en/stable/telegram.bot.html#telegram.Bot.set_my_commands) expects an array of [`BotCommand`](https://docs.python-telegram-bot.org/en/stable/telegram.botcommand.html#telegram.BotCommand) objects.
If Telegram adds new fields to objects that you send as input to Telegram, the corresponding arguments in PTBs classes will be missing.
However, any additional data that is not yet covered by the native Python attributes can be passed via the argument [`api_kwargs`](https://docs.python-telegram-bot.org/en/stable/telegram.telegramobject.html#telegram.TelegramObject.params.api_kwargs) of `TelegramObject` (note that `TelegramObject` is the base class of (almost) all other classes in the `telegram` package).

For example, imagine that Telegram adds a new field called `emoji` to the `BotCommand` class such that this emoji is shown somwhere in the chat when the command is selected by the user. Then passing the argument via `BotCommand("command", "description", emoji="💥")` is possible only after PTB was updated. However, you can already do

```python
BotCommand("command", "description", api_kwargs={"emoji": "💥"})
```

and PTB will pass the data along to Telegram.

> [!Important]
> The argument `api_kwargs` will always accept a dictionary with string keys. Note that the type of the values should in general be either 
>
> * JSON serializable (e.g. `str`, `int`, `list`/`tuple` or `dict` is `str` keys) or
> * a object of type `TelegramObject` (or a subclass) or
> * a `datetime.datetime` object or
> * an [`InputFile`](https://docs.python-telegram-bot.org/en/stable/telegram.inputfile.html) object
> 
> Otherwise, PTB will not be able to correctly pass `api_kwargs` along to Telegram, or Telegram by not be able to parse the input.
> Moreover, convenience functionality provided by PTB for other arguments will not be available for the data passed via `api_kwargs`.
> In particular:
> 
> * insertion of [`Defaults`](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Adding-defaults-to-your-bot) will not work
> * convenience conversion of file handles or file paths into `InputFile` objects will not work

# Using Bot Methods

## New Parameters

All bot methots have a number of parameters defined by Telegram.
If Telegram adds new parameters to mothods that you can specify, the corresponding arguments in PTBs the methods of `telegram.Bot` (and the corresponding shortcuts) will be missing.
However, any additional data that is not yet covered by the native Python arguments can be passed via the argument [`api_kwargs`](https://docs.python-telegram-bot.org/en/stable/telegram.bot.html#telegram.Bot.send_message.params.api_kwargs) of the respective method.

For example, imagine that Telegram adds a new parameter called `delete_after` to the `send_message` method such that the message is deleted automatically after the specified time. Then passing the argument via `await bot.send_message(chat_id=123, text="Hello!", delete_after=42)` is possible only after PTB was updated. However, you can already do

```python
await bot.send_message(chat_id=123, text="Hello!", api_kwargs={"delete_after": 42})
```

and PTB will pass the data along to Telegram.

> [!Important]
> The argument `api_kwargs` of bot methods has the same limitations as the argument `api_kwargs` of `TelegramObject` - see [above](#sending-objects).

## New Methods

Sometimes, Telegram adds entirely new methods to the API that you can use to trigger novel functionality.
If Telegram adds such a new method, the corresponding method of PTBs class `telegram.Bot` will be missing.
Unfurtunately, PTB does not yet have way to make the new method available to you via in a simple way. This feature is being tracked in #4053.
However, you can of course maunally do a HTTP request to the Bot API for the few API calls that are not yet wrapped by PTB.