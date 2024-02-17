# Frequently Asked Questions

*Note:* You may also want to check the official [Telegram Bot FAQ](https://core.telegram.org/bots/faq#what-messages-will-my-bot-get).

- [What messages can my Bot see?](#what-messages-can-my-bot-see)
- [What about messages from other Bots?](#what-about-messages-from-other-bots)
- [Can my bot delete messages from the user in a private chat?](#can-my-bot-delete-messages-from-the-user-in-a-private-chat)
- [How can I get a list of all chats/users/channels my bot is interacting with?](#how-can-i-get-a-list-of-all-chatsuserschannels-my-bot-is-interacting-with)
- [Does my bot get an update, when someone joins my channel?](#does-my-bot-get-an-update-when-someone-joins-my-channel)
- [My bot doesn't receive messages from groups. Why?](#my-bot-doesnt-receive-messages-from-groups-why)
- [Can you add [feature] to PTB? Can I do [thing] with my bot?](#can-you-add-feature-to-ptb-can-i-do-thing-with-my-bot)
- [I'm using `ConversationHandler` and want one handler to be run multiple times. How do I do that?](#im-using-conversationhandler-and-want-one-handler-to-be-run-multiple-times-how-do-i-do-that)
- [I want to handle updates from an external service in addition to the Telegram updates. How do I do that?](#i-want-to-handle-updates-from-an-external-service-in-addition-to-the-telegram-updates-how-do-i-do-that)
- [Why am I getting `ImportError: cannot import name 'XY' from 'telegram'`?](#why-am-i-getting-importerror-cannot-import-name-xy-from-telegram)
- [What do the `per_*` settings in `ConversationHandler` do?](#what-do-the-per_-settings-in-conversationhandler-do)
- [Can I check, if a `ConversationHandler` is currently active for a user?](#can-i-check-if-a-conversationhandler-is-currently-active-for-a-user)
- [How can I list all messages of a particular chat or search through them based on a search query?](#how-can-i-list-all-messages-of-a-particular-chat-or-search-through-them-based-on-a-search-query)
- [Why am I getting an error `The following arguments have not been supplied`?](#why-am-i-getting-an-error-the-following-arguments-have-not-been-supplied)
- [How can I check the version of PTB I am using?](#how-can-i-check-the-version-of-ptb-i-am-using)
- [How do I access info about the message my bot sent?](#how-do-I-access-info-about-the-message-my-bot-sent)
- [How can I print a table in a Telegram message? Is it a lost cause?](#how-can-i-print-a-table-in-a-telegram-message-is-it-a-lost-cause)
- [Can an `InlineKeyboardButton` have both a URL and `callback-data`?](#can-an-inlinekeyboardbutton-have-both-a-url-and-callback-data)
- [Why am I suddenly getting so many log entries from `httpx`?](#why-am-i-suddenly-getting-so-many-log-entries-from-httpx)
- [How can my bot use premium features?](#how-can-my-bot-use-premium-features)

### What messages can my Bot see?

From the official [Telegram Bot FAQ](https://core.telegram.org/bots/faq#what-messages-will-my-bot-get):

> **All bots, regardless of settings, will receive:**
>
> * All service messages.
> * All messages from private chats with users.
> * All messages from channels where they are a member.
>
> **Bot admins and bots with privacy mode disabled will receive all messages except messages sent by other bots.**
> 
> **Bots with privacy mode enabled will receive:**
> 
> * Commands explicitly meant for them (e.g., /command@this_bot).
> * General commands from users (e.g. /start) if the bot was the last bot to send a message to the group.
> * Messages sent via this bot.
> * Replies to any messages implicitly or explicitly meant for this bot.
> 
> **Note that each particular message can only be available to one privacy-enabled bot at a time, i.e., a reply to bot A containing an explicit command for bot B or sent via bot C will only be available to bot A. Replies have the highest priority.**

# ⚠️🚨 **Note:** 🚨⚠️

Turning off the privacy mode has no effect for groups the bot is already in (because obviously that would be a security issue). You need to re-add your bot to those groups.

### What about messages from other Bots?

From the official [Telegram Bot FAQ](https://core.telegram.org/bots/faq#why-doesn-39t-my-bot-see-messages-from-other-bots):
> Bots talking to each other could potentially get stuck in unwelcome loops. To avoid this, we decided that bots will not be able to see messages from other bots regardless of mode.
>
Although, It is still possible to see a bot message if bot is replied to, with `reply_to_message` but you can't do any actions(forward etc) except delete on this message.

### Can my bot delete messages from the user in a private chat?

Yes, but only within the first 48 hours.

### How can I get a list of all chats/users/channels my bot is interacting with?

There is no method for that. You'll need to keep track. See e.g. the [`chatmemberbot.py`](https://docs.python-telegram-bot.org/en/stable/examples.html#examples-chatmemberbot) example.

### Does my bot get an update, when someone joins my channel?

Yes. We receive ChatMemberUpdated update.

### My bot doesn't receive messages from groups. Why?

See [here](#what-messages-can-my-bot-see). TL;DR: Disable group privacy with [@BotFather](https://t.me/BotFather) ⚠️🚨 *and re-add your bot to the group* 🚨⚠️

### Can you add [feature] to PTB? Can I do [thing] with my bot?

Please note that python-telegram-bot is only a *wrapper* for the Telegram Bot API, i.e. PTB can only provide methods that are available through the API.
You can find a full list of all available methods in the [official docs](https://core.telegram.org/bots/api#available-methods).
Anything *not* listed there can not be done with bots. Here is a short list of frequently requested tasks, that can *not* be done with the Bot API:

* Getting a list of all members of a group. You'll need to keep track, e.g. using approaches displayed in [chatmemberbot.py](https://docs.python-telegram-bot.org/en/stable/examples.html#examples-chatmemberbot)
* Adding members to a group/channel (note that you can just send an invite link, which is also less likely to be seen as spam)
* Clearing the chat history for a user
* Getting a message by its `message_id` (For the interested reader: see [here](https://github.com/tdlib/telegram-bot-api/issues/62))
* Getting the last sent message in a chat (you can keep track of that by using [`chat_data`](Storing-bot,-user-and-chat-related-data))
* Getting a users `user_id` via their `@username` (only userbots can do that - you may be interested in [`ptbcontrib/username_to_chat_api`](https://github.com/python-telegram-bot/ptbcontrib/tree/main/ptbcontrib/username_to_chat_api))

In some cases, using a userbot can help overcome restrictions of the Bot API. Please have a look at this [article](http://telegra.ph/How-a-Userbot-superacharges-your-Telegram-Bot-07-09) about userbots.
Note that userbots are *not* what python-telegram-bot is for.

Please also note that some methods marked in the Telegram API (aka MTProto) are marked as "usable for bots". This does *not* necessarily mean that they can be used directly via the Bot API.
See [this GitHub thread](https://github.com/tdlib/telegram-bot-api/issues/1#issuecomment-721873231) and [this discussion](https://github.com/tdlib/telegram-bot-api/issues/62) for more info on that.

### I'm using `ConversationHandler` and want one handler to be run multiple times. How do I do that?

If your handlers callback returns `None` instead of the next state, you will stay in the current state. That means the next incoming update can be handled by the same callback.

### I want to handle updates from an external service in addition to the Telegram updates. How do I do that?

Receiving updates from an external service, e.g. updates about your GitHub repo, is a common use case.
Once you have such an update, you can put them in your bots update queue via `await application.update_queue.put(your_update)`. The `update_queue` is also available as `context.update_queue`.
Note that `your_update` should *not* need to be an instance of `telegram.Update`, as it does not represent an update sent by Telegram. On the contrary, `your_update` can be any type of a Python object. You can e.g. write your own custom class to represent an update from your external service.
To actually do something with the update, you can register a [`TypeHandler`](https://python-telegram-bot.readthedocs.io/telegram.ext.typehandler.html). [`StringCommandHandler`](https://python-telegram-bot.readthedocs.io/telegram.ext.stringcommandhandler.html) and [`StringRegexHandler`](https://python-telegram-bot.readthedocs.io/telegram.ext.stringregexhandler.html) might also be interesting for some use cases.

But how to get the updates into your bot process?
For many cases a simple approach is to check for updates every x seconds. You can use the [`JobQueue`](Extensions---JobQueue) for that.
If you can get the updates via a webhook, you can implement a custom webhook that handles both the Telegram and your custom updates. Please have a look at [`customwebhookbot.py`](https://docs.python-telegram-bot.org/en/stable/examples.html#examples-customwebhookbot) example that showcases how that can be done.
If your third-party service requires some other setup for fetching updates, that surely also be combined with PTB. Keep in mind that you basically only need access to the `(application/context).update_queue`.

### Why am I getting `ImportError: cannot import name 'XY' from 'telegram'`?

There are three common reasons for this kind of exception:

1. You installed `pip install telegram` instead of `pip install python-telegram-bot`. Run `pip uninstall telegram` to uninstall the [telegram library](https://pypi.org/project/telegram/) and then run `pip install python-telegram-bot` again.
2. You have a file named `telegram.py` or a directory/module named `telegram` in your working directory. This leads to namespace issues.
Rename them to something else.
3. You have misconfigured your python path/environment. Please check that the location listed in `pip show python-telegram-bot` is in your [`PYTHONPATH`](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONPATH), or your [`sys.path`](https://docs.python.org/3/library/sys.html#sys.path). If not, then you may for e.g. append it to `PYTHONPATH` with `export PYTHONPATH=$PYTHONPATH:/path/to/python-telegram-bot`, or append it to `sys.path` with `sys.path.append('/path/to/python-telegram-bot')`.

### What do the `per_*` settings in `ConversationHandler` do?

`ConversationHandler` needs to decide somehow to which conversation an update belongs.
The default setting (`per_user=True` and `per_chat=True`) means that in each chat each user can have its own conversation - even in groups.
If you set `per_user=False` and you start a conversation in a group chat, the `ConversationHandler` will also accept input from other users.
Conversely, if `per_user=True`, but `per_chat=False`, its possible to start a conversation in one chat and continue with it in another.

`per_message` is slightly more complicated: Imagine two different conversations, in each of which the user is presented with an inline keyboard with the buttons yes and no.
The user now starts *both* conversations and sees *two* such keyboards. Now, which conversation should handle the update?
In order to clear this issue up, if you set `per_message=True`, the `ConversationHandler` will use the `message_id` of the message with the keyboard.
Note that this approach can only work, if all the handlers in the conversation are `CallbackQueryHandler`s. This is useful for building interactive menus.

**Note:** If you have a `CallbackQueryHandler` in your `ConversationHandler`, you will see a warning `If 'per_message=True/False', …`. It is a *warning*, not an error. If you're sure that you set `per_message` to the correct value, you can just ignore it.
If you like it better, you can even mute with something like

```python
from warnings import filterwarnings
from telegram.warnings import PTBUserWarning

filterwarnings(action="ignore", message=r".*CallbackQueryHandler", category=PTBUserWarning)
```
See [[this page|Exceptions,-Warnings-and-Logging]] for more info.

### Can I check, if a `ConversationHandler` is currently active for a user?

There is no built-in way to do that. You can however easily set a flag as e.g. `context.user_data['in_conversation'] = True` in your `entry_points`s and set it to `False` before returning `ConversationHandler.END`.

### How can I list all messages of a particular chat or search through them based on a search query?

There is no API method for that (see [here](#can-you-add-feature-to-ptb-can-i-do-thing-with-my-bot)). If you really need this functionality, you'll need to save all the messages send to the chat manually. Keep in mind that

1. In group chats your bot doesn't receive all messages, if privacy mode is enabled (see [here](#what-messages-can-my-bot-see))
2. Messages may be edited (in which case your bot will receive a corresponding update)
3. Messages may be deleted (and there are no updates for "message deleted"!)

### Why am I getting an error `The following arguments have not been supplied`?

The `callback` method you pass to `JobQueue.run_*` takes exactly *one* argument, which is of type [`CallbackContext`](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.callbackcontext.html#telegram-ext-callbackcontext). This is, because jobs are triggered by a schedule and not by an update from Telegram. If you want to access data in the callback that changes at runtime (e.g. because you schedule jobs on demand), you can:

1. Access `context.bot_data`.
2. Pass [`{user, chat}_id`](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.jobqueue.html#telegram.ext.JobQueue.run_once.params.chat_id) to any of the `run_*(...)` methods so you can access them in your `callback` as `context.{user, chat}_data`
3. Use `run_*(…, data=additional_data)`. It can then be accessed within the `callback` as `context.job.data`. 

Note that `context.{user, chat}_data` will be `None`, if you don't pass the arguments `{user, chat}_id` to any of the `run_*(...)` methods.

### How can I check the version of PTB I am using?

There are three easy ways to do this. Two work from the command line: `pip show python-telegram-bot` or `python -m telegram`. One you run inside a python script (or the python console): `import telegram`, then call `print(telegram.__version__)`.


### How do I access info about the message my bot sent?

All bot methods have a return value. For example to get the `message_id` of a text message sent by your bot, you can do

```python
message = await bot.send_message(…)
message_id = message.message_id
```

Please check the docs for details about the return value of each bot method.

### How can I print a table in a Telegram message? Is it a lost cause?

Long story short: yes, it's a lost cause.
Telegram formatting doesn't support tables and even if you try to get everything aligned with whitespaces and tabs, there WILL be a client that has a different max-widths for the text bubbles or a different font/font size and everything will be messed up. 
If it's important to you to send a nicely formatted table, send a picture or a pdf.

### Can an `InlineKeyboardButton` have both a URL and `callback-data`?

No, exactly *one* of the optional arguments of `InlineKeyboardButton` must be set.
The closest that you can get to having both a URL and `callback_data` in the button is:

1. have a custom server (e.g. `my.tld`) where you can create redirect-links on the fly - something similar to bitly or all the other link shortening services
2. each time you want to have both a URL and a `callback_data`, create a new link `my.tld/some_token`
    1. Make `my.tld/some_token` redirect to the actual URL
    2. Configure your server such that it sends a notification to your bot telling it that the `my.tld/some_token` was accessed
3. Make your bot process that information similar to how you'd process a `CallbackQuery`. See also [this FAQ entry](#i-want-to-handle-updates-from-an-external-service-in-addition-to-the-telegram-updates-how-do-i-do-that)

### Why am I suddenly getting so many log entries from `httpx`?

Starting with [v.0.24.1](https://github.com/encode/httpx/releases/tag/0.24.1), `httpx` logs all async requests at `INFO` level, which may be annoying for you as a PTB user.

You can explicitly set logging level for `httpx` to `WARNING` to get rid of these messages:
```py
import logging

logging.getLogger("httpx").setLevel(logging.WARNING)
```

### How can my bot use premium features?

Currently, Bot API allows only limited premium features to be used by bots, particularly sending custom emojis.
Custom emoji entities can only be used by bots that purchased additional usernames on [Fragment](https://fragment.com/) in any chat.
Additionally, bots can send custom emojis in specific groups without buying additional usernames if that group has sufficient amount of boosts to allow non premium members to send custom emojis from the groups emoji pack. See [formatting options](https://core.telegram.org/bots/api#formatting-options) to know how to send custom emojis if your bot fulfills the necessary conditions.