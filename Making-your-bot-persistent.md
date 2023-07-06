In V12.0b1 we added a persistence mechanism to `telegram.ext`. This wiki page is there to help you understand and set up persistence for your bots.

- [What can become persistent?](#what-can-become-persistent-)
- [Included persistence classes](#included-persistence-classes)
- [3rd party persistence classes](#3rd-party-persistence-classes)
- [What do I need to change?](#what-do-i-need-to-change)
- [Refreshing at runtime](#refreshing-at-runtime)
- [Storing Bots](#storing-bots)

## What can become persistent?
* The persistence structure is designed to make
  * `bot_data`
  * `chat_data`
  * `user_data`,
  * `ConversationHandler`'s states and
  * `ExtBot.callback_data_cache` persistent.

* `Job`'s and the `job_queue` is not supported.
However, the current `JobQueue` backend [APScheduler](https://apscheduler.readthedocs.io/) has its own persistence logic that you can leverage.
See e.g. [`ptbcontrib/ptb_sqlalchemy_jobstore`](https://github.com/python-telegram-bot/ptbcontrib/tree/main/ptbcontrib/ptb_sqlalchemy_jobstore)
* For a special note about `Bot` instances, see [below](#storing-bots)

## Included persistence classes
Three classes concerning persistence in bots have been added.  
* [BasePersistence](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.basepersistence.html) - Is an interface class for persistence classes.
If you create your own persistence classes to maintain a database-connection for example, you must inherit from `BasePersistence` and implement all abstract methods
* [PicklePersistence](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.picklepersistence.html) - Uses pickle files to make the bot persistent.  
* [DictPersistence](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.dictpersistence.html) - Uses in memory dicts and easy conversion to and from JSON to make the bot persistent.
Note that this class is mainly intended as starting point for custom persistence classes that need to JSON-serialize the stored data before writing them to file/database and does *not* actually write any data to file/database.

## 3rd party persistence classes
Instead of manually handling a database to store data, consider implementing a subclass of `BasePersistence`. This allows you to simply pass an instance of that subclass to the `Application` and let PTB handle the loading, updating & storing of the data!

If you want to create your own persistence class, please carefully read the docs on [BasePersistence](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.basepersistence.html). It will tell you what methods you need to overwrite. 

If you've written a persistence class that could benefit others (e.g., a general one covering all types of data), it would be great if you linked it here or even better made it available in [ptbcontrib](https://github.com/python-telegram-bot/ptbcontrib).

These 3rd party packages contain persistence classes (the list is incomplete):
* [python-telegram-bot-django-persistence](https://github.com/GamePad64/python-telegram-bot-django-persistence) - Uses Django ORM to store persistence data. It is most useful for projects, that use PTB and Django.
* [MongoPersistence](https://github.com/LucaSforza/MongoPersistence) - Package to add persistence to your telegram bot and upload data to your mongo database.

## What do I need to change?

To make your bot persistent you need to do the following.

- Create a persistence object (e.g. `my_persistence = PicklePersistence(filepath='my_file')`)
- Construct `Application` with the persistence (`Application.builder().token('TOKEN').persistence(persistence=my_persistence).build()`).

This is enough to make `user_data`, `bot_data`, `chat_data` and `ExtBot.callback_data_cache` persistent.
To make a conversation handler persistent (save states between bot restarts) you **must name it** and set `persistent` to `True`.
For example `ConversationHandler(..., persistent=True, name='my_name')`. `persistent` is `False` by default.
Adding these arguments and adding the conversation handler to a persistence-aware `Application` will make it persistent.

When starting the `Application` with `Application.start()` or `Application.run_{polling, webhook}`, it will loads the data from the persistence on startup and automatically update the persistence in regular intervals.
You can customize the interval via the [`update_interval`](https://python-telegram-bot.readthedocs.io/telegram.ext.basepersistence.html#telegram.ext.BasePersistence.params.update_interval) argument of `Base/Pickle/Dict/…Persistence`.

### ⚠️ Note

Since the persisted data is loaded on start-up, any data written to `Application.{bot, chat, user_data}` *before* startup will hence be overridden! To manually write data into these *after* the persisted data has been loaded, please use [`Application.post_init`](https://docs.python-telegram-bot.org/telegram.ext.applicationbuilder.html?highlight=ApplicationBuilder#telegram.ext.ApplicationBuilder.post_init).

## Refreshing at runtime

If your persistence reads the data from an external database, the entries in this database could change at runtime.
This is the case in particular, if the entries in the database are created by a 3rd party service independently of your bot.
If you want to make sure that the data in `context.user/chat/bot_data` are always up-to-date, your persistence class should implement the methods [`refresh_bot/chat/user_data`](https://python-telegram-bot.readthedocs.io/telegram.ext.basepersistence.html#telegram.ext.BasePersistence.refresh_chat_data).
Those will be called when in update comes in, before any of your callbacks are called.

These methods can also be useful to implement a lazy-loading strategy.

## Storing Bots

Instances of `telegram.Bot` should not be serialized, because changes to the bot don't apply to the serialized object.

For example, you might change the [[default values|Adding-defaults-to-your-bot]] used by the bot.
Or if you change the bots token, this may lead to e.g. `Chat not found` errors.
This is relevant e.g., if you store Telegram objects like `Message` in `bot/user/chat_data`, as some of them hold a reference to `Application.bot` (which is how the shortcuts like `Message.reply_text` work).

The interface class `BasePersistence` does not question what kind of data you supply to its methods.
Hence, each implementation should take care that it does not try to serialize `telegram.Bot` instances.
For example, it can check if the data equals the attribute `BasePersistence.bot` (which will be the bot object used by the `Application`) and instead store a placeholder.
When loading the data, the `BasePersistence.bot` can be reinserted instead of the placeholder.
Indeed, this is basically what the built-in `PicklePersistence` does.

For more technical details, please refer to the documentation of [`BasePersistence`](https://python-telegram-bot.readthedocs.io/telegram.ext.basepersistence.html#telegram-ext-basepersistence), 
[`PicklePersistence`](https://python-telegram-bot.readthedocs.io/telegram.ext.picklepersistence.html#telegram-ext-picklepersistence)

### ⚠️ Note
Although `PicklePersistence` does the 'placeholder' process described above, all the data are deep copied with `copy.deepcopy` before being handed over to persistence. This means that you should either store only copyable data (e.g. no `telegram.Bot` objects) and/or ensure that your stored data defines appropriate custom deepcopy behavior. This technical detail is described in a note [here](https://docs.python-telegram-bot.org/telegram.ext.application.html#telegram.ext.Application.update_persistence)