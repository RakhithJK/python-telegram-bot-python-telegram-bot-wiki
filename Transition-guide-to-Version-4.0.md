# What's new?
All changes can also be reviewed in our [official documentation](http://python-telegram-bot.readthedocs.io/)!
## Dispatcher

**For users coming from RC release predating 26th of April, 2016**: 
- Changes in "filters" syntax (upper/lower case)
- Handler groups are now identified by `int` only, and are processed in order, smallest first.

The `Dispatcher` class has now a cleaner interface and more precise Message filtering. Instead of many methods with long names like `Dispatcher.addTelegramMessageHandler(handler)`, we now only have two of those methods:

> #### `add_handler(handler, group=0)`
> Register a handler.
>
> TL;DR: Order and priority counts. 0 or 1 handlers per group will be used.
>
> A handler must be an instance of a subclass of `telegram.ext.Handler`. All handlers are organized in groups with a numeric value. The default group is `0`. All groups will be evaluated for handling an update, but only 0 or 1 handler per group will be used.
>
> The priority/order of handlers is determined as follows:
>
> - Priority of the group (lower group number == higher priority)
> - The first handler in a group which should handle an update will be used. Other handlers from the group will not be used. The order in which handlers were added to the group defines the priority.

> ##### Parameters:
> `handler (Handler)` - A Handler instance  
> `group (optional[int])` - The group identifier. Default is `0`

> #### `add_error_handler(callback)`

> This method remains unchanged, only the name has been changed to snake_case.

So, the `add_handler` method is accepting an object of a subclass of `telegram.ext.Handler`. Let's see how that looks in real life:

```python
from telegram.ext import MessageHandler, Filters

def text_callback(bot, update):
  print("New text message: " + update.message.text)

dispatcher.add_handler(MessageHandler([Filters.text], text_callback))
```

As you can see here, the `MessageHandler` class is one of the included `Handler` subclasses. All that was possible before is still possible, but now more organized and more explicit. Lets take a quick look at another handler class, the `RegexHandler`:

```python
from telegram.ext import RegexHandler

def name_callback(bot, update, groupdict):
  print("The name of the user is: " + groupdict['name'])

name_regex = r'My name is (?P<name>.*)'
dispatcher.add_handler(RegexHandler(name_regex, text_callback, pass_groupdict=True))
```
Here you can see the optional argument `groupdict` passed to the handler callback function. Note that it is necessary to specify this explicitly when creating the `Handler` object.

### Other changes
* You can easily implement your own handlers. Just subclass `telegram.ext.handler` and take a look at the implementation of the provided handlers.
* Instead of `addTelegramInlineHandler` there are now `InlineHandler`, `ChosenInlineResultHandler` and `CallbackQueryHandler`
* There is no replacement for `addUnknownTelegramCommandHandler`. Instead, it is recommended to use `RegexHandler(r'/.*', ...)` and add it as the last handler
* The `UpdateQueue` class and `context` parameters have been removed

## Bot API 2.0

Please read the documentation of the [Telegram Bot API](https://core.telegram.org/bots/api#recent-changes) to learn about all the new things in version 2 of the bot API. This section covers only those changes that are not backwards compatible and not listed in the **Recent Changes** list.

* `new_chat_participant` and `left_chat_participant` of the `Message` class are now `new_chat_member` and `left_chat_member`
* The following parameters on `InlineResult` and `InlineQueryResult` objects are removed in favor of `InlineMessageContent`:
 - message_text
 - parse_mode
 - disable_web_page_preview
* In `InlineQueryResultPhoto` the parameter `mime_type` has been removed. JPEG is now required.
* `ReplyKeyboardMarkup` now takes a list of a list of `KeyboardButton` instead of strings.
 - From v4.0.2 you can use `str` again

## The `telegram.ext` module

The classes `Updater`, `Dispatcher` and `JobQueue` that were previously available for import directly `from telegram` are now located in `telegram.ext`.