This page is a collection of sorts, dedicated to showcase design patterns we get asked about often in our support group.

- [Requirements](#requirements)
- [How to handle updates in several handlers](#how-to-handle-updates-in-several-handlers)
  - [Type Handler and Groups](#type-handler-and-groups)
  - [Boilerplate Code](#boilerplate-code)
    - [But I do not want a handler to stop other handlers](#but-i-do-not-want-a-handler-to-stop-other-handlers)
  - [How do I limit who can use my bot?](#how-do-i-limit-who-can-use-my-bot)
  - [How do I rate limit users of my bot?](#how-do-i-rate-limit-users-of-my-bot)
  - [Conclusion](#conclusion)
- [How do I enforce users joining a specific channel before using my bot?](#how-do-i-enforce-users-joining-a-specific-channel-before-using-my-bot)
- [How do I send a message to all users of the bot?](#how-do-i-send-a-message-to-all-users-of-the-bot)
- [How do I deal with a media group?](#how-do-i-deal-with-a-media-group)
  - [Timer based approach](#timer-based-approach)
  - [Manual approach](#manual-approach)
- [Running PTB alongside other `asyncio` frameworks](#running-ptb-alongside-other-asyncio-frameworks)
- [How to deal with multiple CBQ from one button](#how-to-deal-with-multiple-CBQ-from-one-button)

## Requirements

Knowing how to make bots with PTB is enough. That means you should be familiar with Python and with PTB.
If you haven't worked on anything with PTB, then please check [Introduction to the API](../Introduction-to-the-API) as well as the [Tutorial: Your first Bot](../Extensions---Your-first-Bot).

## How to handle updates in several handlers

At some point developing one's bots, most of us face the following question

> How do I handle an update _before_ other handlers?

<!-- Im sorry, I love the section, but I don't think it fits in the wiki site, because it is designed a bit more dense. Sorry!
This guide is written as a kick-starter to help you in tackling the above mentioned and similar use cases.
If you are looking an answer for:

- How to prevent a set of users/groups from accessing my bot?
- How to control flooding of my bot?
- I want my bot to process every update in addition to other handlers. How can I do it?
Then this guide will hint you a possible solution.
-->

The following sections will give you an idea how to tackle this problem, based on frequent scenarios where this problem arises.

### Type Handler and Groups

PTB comes with a powerful handler known as [TypeHandler](https://python-telegram-bot.readthedocs.io/telegram.ext.typehandler.html).
You can understand it as a generic handler. You can use it to handle any class put through the Updater.
For example, Type Handlers are used in bots to handle "updates" from Github or other external services.

To add any handler, we use [Application.add_handler](https://python-telegram-bot.readthedocs.io/telegram.ext.application.html#telegram.ext.Application.add_handler). Apart from the handler itself, it takes an optional argument called `group`. We can understand groups as numbers which indicate the priority of handlers. A lower group means a higher priority. An update can be processed by (at most) one handler in each group.

Stopping handlers in higher groups from processing an update is achieved using [ApplicationHandlerStop](https://python-telegram-bot.readthedocs.io/telegram.ext.ApplicationHandlerStop.html#telegram.ext.ApplicationHandlerStop). When raising this exception, the Application is asked to stop sending the updates to handlers in higher groups. Depending on your use case, you may not need to raise it. But it is useful if you want to enable flood handling or limit who can use the bot.

That's it. With these three knowledge nuggets, we can solve the question given in the introduction.

### Boilerplate Code

Before working on the problems, we will provide you with a template of code that you can use. All you need to do to follow this guide is change the internals of your `callback` and `group` as required by the problem you face.

```python
from telegram import Update
from telegram.ext import ContextTypes, ApplicationHandlerStop, TypeHandler, Application


async def callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle the update"""
    await do_something_with_this_update(update, context)
    raise ApplicationHandlerStop # Only if you DON'T want other handlers to handle this update


app = Application.builder().token("TOKEN").build()
handler = TypeHandler(Update, callback) # Making a handler for the type Update
app.add_handler(handler, -1) # Default is 0, so we are giving it a number below 0
# Add other handlers and start your bot.
```

The code above should be self-explanatory, provided you read the previous section along with the respective documentation. We made a handler for `telegram.Update` and added it to a lower group.

#### But I do not want a handler to stop other handlers

In case you don't want to stop other handlers from processing the update, then you should modify your `callback` to not raise the exception.
This is a generic use case often used for analytics purpose. For example, if you need to add every user who uses your bot to a database, you can use this method. Simply put, this sort of approach is used to keep track of every update.

```python
async def callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    add_message_to_my_analytics(update.effective_message)
    add_user_to_my_database(update.effective_user)
```

Note the difference in this example compared to the previous ones. Here we don't raise `ApplicationHandlerStop`. These type of handlers are known as _shallow handlers_ or _silent handlers_. These type of handlers handle the update and also allow it to be handled by other common handlers like `CommandHandler` or `MessageHandler`. In other words, they don't block the other handlers.

Now let us solve the specific use cases. All you need to do is modify your `callback` as required. 😉

### How do I limit who can use my bot?

> Please be sure to start reading this section from the [top](#how-to-handle-updates-in-several-handlers).

To restrict your bot to a set of users or if you don't want it to be available for a specific group of people, you can use a `callback` similar to the following. Remember, the process is same if you want to enable/disable the bot for groups or channels.

```python
SPECIAL_USERS = [127376448, 172380183, 1827979793]  # Allows users

async def callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.user_id in SPECIAL_USERS:
        pass
    else:
        await update.effective_message.reply_text("Hey! You are not allowed to use me!")
        raise ApplicationHandlerStop
```

Here, it should be noted that this approach blocks your bot entirely for a set of users. If all you need is to block a specific functionality, like a special command or privilege, then it will be wise to use [filters.Chat](https://python-telegram-bot.readthedocs.io/telegram.ext.filters.html#telegram.ext.filters.Chat), [filters.User](https://python-telegram-bot.readthedocs.io/telegram.ext.filters.html#telegram.ext.filters.User).
Don't forget that you can also use [decorators](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Code-snippets#restrict-access-to-a-handler-decorator) or a simple `if-else` check.
If you want a more streamlined style of managing permissions (like superuser, admin, users) then [ptbcontrib/roles](https://github.com/python-telegram-bot/ptbcontrib/tree/main/ptbcontrib/roles) is worth checking out.

### How do I rate limit users of my bot?

> Please be sure to start reading this section from the [top](#how-to-handle-updates-in-several-handlers).

The exact definition of _rate limit_ depends on your point of view. You typically should keep record of previous usage of the user and warn them when they cross a limit. Here, for demonstration, we use a method that restricts the usage of the bot for 5 minutes.

```python
from time import time

MAX_USAGE = 5


async def callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    count = context.user_data.get("usageCount", 0)
    restrict_since = context.user_data.get("restrictSince", 0)

    if restrict_since:
        if (time() - restrict_since) >= 60 * 5: # 5 minutes
            del context.user_data["restrictSince"]
            del context.user_data["usageCount"]
            await update.effective_message.reply_text("I have unrestricted you. Please behave well.")
        else:
            await update.effective_message.reply_text("Back off! Wait for your restriction to expire...")
            raise ApplicationHandlerStop
    else:
        if count == MAX_USAGE:
            context.user_data["restrictSince"] = time()
            await update.effective_message.reply_text("Stop flooding! Don't bother me for 5 minutes...")
            raise ApplicationHandlerStop
        else:
            context.user_data["usageCount"] = count + 1
```

The approach we used is dead lazy. We keep a count of updates from the user and when it reaches maximum limit, we note the time. We proceed to stop handling the updates of that user for 5 minutes. Your effective flood limit strategy and punishment may vary. But the logic remains same.

### Conclusion

We have seen how `TypeHandler` can be used to give a fluent experience without messing up our code-base. Now you would be able to solve complex use cases from the given examples. But please note that `TypeHandler` **is not** the only option.
If you feel like this approach is too much of trouble, you can use Python's inbuilt [decorators](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Code-snippets#restrict-access-to-a-handler-decorator).

## How do I enforce users joining a specific channel before using my bot?

After sending an (invite) link to the channel to the user, you can use [`Bot.get_chat_member`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.get_chat_member) to check if the user is an that channel.
Note that:

- the bot needs to be admin in that channel
- the user must have started the bot for this approach to work. If you try to run `get_chat_member` for a user that has not started the bot, the bot can not find the user in a chat, even if it is a member of it.

Otherwise depending on whether the user in the channel, has joined and left again, has been banned, ... (there are multiple situations possible), the method may

- raise an exception and in this case the error message will probably be helpful
- return a [`ChatMember`](https://python-telegram-bot.readthedocs.io/telegram.chatmember.html#telegram.ChatMember) instance. In that case make sure to check the [`ChatMember.status`](https://python-telegram-bot.readthedocs.io/telegram.chatmember.html#telegram.ChatMember.status) attribute

Since API 5.1 (PTB v13.4+) you can alternatively use the [`ChatMember`](https://python-telegram-bot.readthedocs.io/telegram.chatmemberupdated.html) updates to keep track of users in channels. See [`chatmemberbot.py`](https://docs.python-telegram-bot.org/examples.html#examples-chatmemberbot) for an example.

If the user has not yet joined the channel, you can ignore incoming updates from that user or reply to them with a corresponding warning. A convenient way to do that is by using [TypeHandler](https://python-telegram-bot.readthedocs.io/telegram.ext.typehandler.html). Read this [section](#how-do-i-limit-who-can-use-my-bot) to learn how to do it.

## How do I send a message to all users of the bot?

Let's first point out an easy alternative solution: Instead of sending the messages directly through your bot, you can instead set up a channel to publish the announcements. You can link your users to the channel in a welcome message.

If that doesn't work for you, here we go:

To send a message to all users, you of course need the IDs of all the users. You'll have to keep track of those yourself. The most reliable way for that are the [`my_chat_member`](https://python-telegram-bot.readthedocs.io/telegram.chatmemberupdated.html) updates. See [`chatmemberbot.py`](https://docs.python-telegram-bot.org/examples.html#examples-chatmemberbot) for an example on how to use them.

If you didn't keep track of your users from the beginning, you may have a chance to get the IDs anyway, if you're using persistence. Please have a look at [this issue](https://github.com/python-telegram-bot/python-telegram-bot/issues/1836) in that case.

Even if you have all the IDs, you can't know if a user has blocked your bot in the meantime. Therefore, you should make sure to wrap your send request in a `try-except` clause checking for [`telegram.error.Forbidden`](https://python-telegram-bot.readthedocs.io/telegram.error.html#telegram.error.Forbidden) errors.

Finally, note that Telegram imposes some limits that restrict you to send ~30 Messages per second. If you have a huge user base and try to notify them all at once, you will get flooding errors. To prevent that, try spreading the messages over a long time range. To achieve that you can use e.g.

* the [`JobQueue`](../wiki/Extensions---JobQueue)
* PTBs mechanism to [avoid flood limits](../wiki/Avoiding-flood-limits)

## How do I deal with a media group?

The basic problem behind this question is simple. For the end user, it looks like one message, consisting of several medias, are sent to the receiver. For the bot API/bot developer, this is not the case however: Every media is send as one unique message, only linked via the unique [Message.media_group_id](https://python-telegram-bot.readthedocs.io/telegram.message.html#telegram.Message.media_group_id) attribute. So you need some way of determining when to start and to end collecting the media messages.

This basic problem has two basic approaches for handling it, without requiring a more elaborate setup involving databases.

### Timer based approach

[Real life code example.](https://github.com/Poolitzer/channelforwarder/blob/589104b8a808199ba46d620736bd8bea1dc187d9/main.py#L19-L46)

This approach has the upside of looking seamless to the user. The downside is that there is a (low) possibility that one part of the media group is missed.

The idea behind this approach is to start a timer (and an array with the message object/id in it) when receiving the first media_group message. Until the timer runs out, every incoming message with the same media id will be added to the array. Once the timer runs out, the media group is considered done and can be dealt with according to your situation.

There is a possibility that a part of the media group is received after the timer ran out. This could be because the network was too slow to deliver the updates to you (more likely with a webhook setup since long polling can receive multiple updates at once) or your server took to long to deal with the update (or a combination of both). The result of this happening need to be determined by you.

### Manual approach

```diff
- Real life code example TBD
```

This approach has two upsides: You don't force users to use media groups (so they can e.g. send more media then fits in one group) and you will not miss a media. The downside is that it requires manual interaction from users, and we all know how special users can be :)

The idea behind this approach is to start a upload segment in your code. Either in a `ConversationHandler` or with a `CommandHandler`. Ask the user then to send all medias. Once they send the first (or the first media group, this doesn't matter in your code), you store the information you need in an array. Then you ask them to send either more or send a command like `/finish` / a specific text message you intercept with a [MessageHandler + regex Filter](../Types-of-Handlers#pattern-matching-filtersregex). The point behind this is to have the user finish the addition of media on their own terms. Once they triggered the second handler, you can consider the array finished.

## Running PTB alongside other `asyncio` frameworks

The [tutorial](../Extensions---Your-first-Bot) as well (almost) all the [examples](https://docs.python-telegram-bot.org/examples.html#examples-chatmemberbot) make use of [`Application.run_polling`](https://docs.python-telegram-bot.org/telegram.ext.application.html#telegram.ext.Application.run_polling).
This method is blocking, which means that no other `asyncio` related code can be started while it is running.
This is okay as long your Python script runs only your bot.
However, if you want to run multiple bots in the same Python script or other `asyncio` frameworks (e.g. a webserver) alongside your bot, this becomes an issue.

The `Application` class was designed with these use cases in mind and `Application.run_polling` can be understood as mainly a  convenience method for the important use case of running "just" a single bot.
The same holds for `Application.run_webhook.`

Without using `Application.run_{webhook, polling}`, the overall logic of startup and shutdown of the `Application` is as follows:

```python
application = ApplicationBuilder().token("TOKEN").build()

async def main():
    await application.initialize()
    await application.start()
    await application.updater.start_{webhook, polling}()
    # Start other asyncio frameworks here
    # Add some logic that keeps the event loop running until you want to shutdown
    # Stop the other asyncio frameworks here
    await application.updater.stop()
    await application.stop()
    await application.shutdown()
```

Several things to note here:

* Of course, the "other `asyncio` framework" could be another `Application`, i.e. a second bot that you want to run in the same Python script.
* The important part is that `Application.initialize`, `Application.start`, `Updater.start_{webhook, polling}`, `Application.stop` and `Application.shutdown` are called in the shown order. How exactly this is done is up to you. E.g. it may be beneficial to use `loop.run_until_complete` instead of `await`-ing the coroutines.
* Instead of calling `Application.{initialize, shutdown}`, you can also use the application as a context manager, i.e.
  ```python
  application = ApplicationBuilder().token("TOKEN").build()
  
  async def main():
      async with application:  # Calls `initialize` and `shutdown`
          await application.start()
          await application.updater.start_{webhook, polling}()
          # Start other asyncio frameworks here
          # Add some logic that keeps the event loop running until you want to shutdown
          # Stop the other asyncio frameworks here
          await application.updater.stop()
          await application.stop()
  ```
* Clean startup, execution and shutdown of `asyncio` processes is not a trivial topic, there are many approaches to this and probably just as many opinions on which is the best.
  Covering this topic is out of scope for this wiki.
  The important point however is that there is not a single "right" way to do this - for PTB or in general.
  In particular the part where you have to keep the event loop running can take differnt form depending on your use case, your personal preferences and the frameworks you use.
  If you are interested, we invite you to have a look at the [source code](https://github.com/python-telegram-bot/python-telegram-bot/blob/master/telegram/ext/_application.py) of the `Application.run_{webhook, polling}` methods to see how PTB handles this.
* Shutdown logic of `asyncio` processes may involve any pending `asyncio.Tasks` by cancelling all tasks returned by `asyncio.all_tasks`. This should be done only *after* `Application.stop` was called, since `Application.start` starts `asyncio.Tasks` in the background that should be allowed to finish.
* Calling `application.updater.start_{webhook, polling}` is not mandatory.
  In fact, using PTBs `Updater` for fetching updates from Telegram is optional, and you can use a custom implementation instead, if you like (see [[this wiki page|Architecture]] for details).
  The [`customwebhookbot`](https://docs.python-telegram-bot.org/examples.html#examples-customwebhookbot) example showcases this use case, which also involves manually starting and stopping the application. Keeping the event loop running is covered by the `uvicorn` framework in this example.

## How to deal with multiple CBQ from one button
A common design is to have an inline keyboard and on pressing the button, edit the message to have this button vanish. Short example:

```python
import asyncio
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import Application, CallbackQueryHandler, CommandHandler, ContextTypes


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Sends a message with an inline buttons attached."""
    keyboard = [[InlineKeyboardButton("Button To Vanish", callback_data="1")]]
    await update.message.reply_text(
        "Spam it", reply_markup=InlineKeyboardMarkup(keyboard)
    )


async def button(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Parses the CallbackQuery and updates the message text."""
    query = update.callback_query
    await query.answer()
    # long wait for something
    await asyncio.sleep(5)
    await query.edit_message_text(text=f"You did it")


def main() -> None:
    """Run the bot."""
    application = Application.builder().token("TOKEN").build()
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CallbackQueryHandler(button))
    application.run_polling()


if __name__ == "__main__":
    main()

```
As you can see, the button has to wait for 5 seconds, which allows impatient users to spam the button. This results in multiple callback query updates from this button, even though (without the sleep call) you might expect just one CBQ update from it. All "duplicate" updates from the button will result in a _BadRequest_, because the message content is the same one as the previous (it was already edit to "You did it" and the code tries to edit it to the same text again).

The solution to this problem is to keep track independently if the message was already edited. Thanks to user data, this is achieved quite easily:
```python

async def button(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Parses the CallbackQuery and updates the message text."""
    query = update.callback_query
    unique_data = query.data + (
        query.inline_message_id
        if query.inline_message_id
        else str(query.message.message_id)
    )
    if "last_cbq" not in context.user_data:
        context.user_data["last_cbq"] = unique_data
    else:
        if context.user_data["last_cbq"] == unique_data:
            return
    await query.answer()
    # long wait for something
    await asyncio.sleep(5)
    await query.edit_message_text(text=f"You did it")
```
This setup has two downsides:
1. You have to write this in the beginning of each CBQ handler. You can either use a [decorators](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Code-snippets#restrict-access-to-a-handler-decorator) or a [separate handler](#how-to-handle-updates-in-several-handlers) to avoid this.
1. You can not edit in a button with the same callback data as the previous one in the same message.

If you remember the last point, you will not face any issues with this solution however.