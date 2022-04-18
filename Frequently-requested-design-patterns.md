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

## Requirements

Knowing how to make bots with PTB is enough. That means you should be familiar with Python and with PTB.
If you haven't worked on anything with PTB, then please check [Introduction to the API](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Introduction-to-the-API).

## How to handle updates in several handlers

At some point developing ones bots, most of us face the following question

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

PTB comes with a powerful handler known as [TypeHandler](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.typehandler.html).
You can understand it as a generic handler. You can use it to handle any class put through the Updater.
For example, Type Handlers are used in bots to handle "updates" from Github or other external services.

To add any handler, we use [Dispatcher.add_handler](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.dispatcher.html#telegram.ext.Dispatcher.add_handler). Apart from the handler itself, it takes an optional argument called `group`. We can understand groups as numbers which indicate the priority of handlers. A lower group means a higher priority. An update can be processed by (at most) one handler in each group.

Stopping handlers in higher groups from processing an update is achieved using [DispatcherHandlerStop](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.dispatcherhandlerstop.html#telegram.ext.DispatcherHandlerStop). When raising this exception, the Dispatcher is asked to stop sending the updates to handlers in higher groups. Depending on your use case, you may not need to raise it. But it is useful if you want to enable flood handling or limit who can use the bot.

That's it. With these three knowledge nuggets, we can solve the question given in the introduction.

### Boilerplate Code

Before working on the problems, we will provide you with a template of code that you can use. All you need to do to follow this guide is change the internals of your `callback` and `group` as required by the problem you face.

```python
from telegram import Update
from telegram.ext import CallbackContext, DispatcherHandlerStop, TypeHandler, Updater


def callback(update: Update, context: CallbackContext):
    """Handle the update"""
    do_something_with_this_update(update, context)
    raise DispatcherHandlerStop # Only if you DON'T want other handlers to handle this update


updater = Updater(TOKEN)
dispatcher = updater.dispatcher
handler = TypeHandler(Update, callback) # Making a handler for the type Update
dispatcher.add_handler(handler, -1) # Default is 0, so we are giving it a number below 0
# Add other handlers and start your bot.
```

The code above should be self-explanatory, provided you read the previous section along with the respective documentation. We made a handler for `telegram.Update` and added it to a lower group.

#### But I do not want a handler to stop other handlers

In case you don't want to stop other handlers from processing the update, then you should modify your `callback` to not raise the exception.
This is a generic use case often used for analytics purpose. For example, if you need to add every user who uses your bot to a database, you can use this method. Simply put, this sort of approach is used to keep track of every update.

```python
def callback(update: Update, context: CallbackContext):
    add_message_to_my_analytics(update.effective_message)
    add_user_to_my_database(update.effective_user)
```

Note the difference in this example compared to the previous ones. Here we don't raise `DispatcherHandlerStop`. This type of handlers is known as _shallow handler_ or _silent handler_. These type of handlers handle the update and also allow it to be handled by other common handlers like `CommandHandler` or `MessageHandler`. In other words, they don't block the other handlers.

Now let us solve the specific use cases. All you need to do is modify your `callback` as required. 😉

### How do I limit who can use my bot?

To restrict your bot to a set of users or if you don't want it to be available for a specific group of people, you can use a `callback` similar to the following. Remember, the process is same if you want to enable/disable the bot for groups or channels.

```python
SPECIAL_USERS = [127376448, 172380183, 1827979793] # Allows users

def callback(update: Update, context: CallbackContext):
    if update.effective_user.user_id in SPECIAL_USERS:
        pass
    else:
        update.effective_message.reply_text("Hey! You are not allowed to use me!")
        raise DispatcherHandlerStop
```

Here, it should be noted that this approach blocks your bot entirely for a set of users. If all you need is to block a specific functionality, like a special command or privilege, then it will be wise to use [Filters.chat](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.filters.html#telegram.ext.filters.Filters.chat), [Filters.user](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.filters.html#telegram.ext.filters.Filters.user).
Don't forget that you can also use [decorators](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Code-snippets#restrict-access-to-a-handler-decorator) or a simple `if-else` check.
If you want a more streamlined style of managing permissions (like superuser, admin, users) then [ptbcontrib/roles](https://github.com/python-telegram-bot/ptbcontrib/tree/main/ptbcontrib/roles) is worth checking out.

### How do I rate limit users of my bot?

The exact definition of _rate limit_ depends on your point of view. You typically should keep record of previous usage of the user and warn them when they cross a limit. Here, for demonstration, we use a method that restricts the usage of the bot for 5 minutes.

```python
from time import time

MAX_USAGE = 5


def callback(update: Update, context: CallbackContext):
    count = context.user_data.get("usageCount", 0)
    restrict_since = context.user_data.get("restrictSince", 0)

    if restrict_since:
        if (time() - restrict_since) >= 60 * 5: # 5 minutes
            del context.user_data["restrictSince"]
            del context.user_data["usageCount"]
            update.effective_message.reply_text("I have unrestricted you. Please behave well.")
        else:
            update.effective_message.reply_text("Back off! Wait for your restriction to expire...")
            raise DispatcherHandlerStop
    else:
        if count == MAX_USAGE:
            context.user_data["restrictSince"] = time()
            update.effective_message.reply_text("Stop flooding! Don't bother me for 5 minutes...")
            raise DispatcherHandlerStop
        else:
            context.user_data["usageCount"] = count + 1
```

The approach we used is dead lazy. We keep a count of updates from the user and when it reaches maximum limit, we note the time. We proceed to stop handling the updates of that user for 5 minutes. Your effective flood limit strategy and punishment may vary. But the logic remains same.

### Conclusion

We have seen how `TypeHandler` can be used to give a fluent experience without messing up our code-base. Now you would be able to solve complex use cases from the given examples. But please note that `TypeHandler` **is not** the only option.
If you feel like this approach is too much of trouble, you can use Python's inbuilt [decorators](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Code-snippets#restrict-access-to-a-handler-decorator).

## How do I enforce users joining a specific channel before using my bot?

After sending an (invite) link to the channel to the user, you can use [`Bot.get_chat_member`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.bot.html#telegram.Bot.get_chat_member) to check if the user is an that channel.
Note that:

- the bot needs to be admin in that channel
- the user must have started the bot for this approach to work. If you try to run `get_chat_member` for a user that has not started the bot, the bot can not find the user in a chat, even if it is a member of it.

Otherwise depending on whether the user in the channel, has joined and left again, has been banned, ... (there are multiple situations possible), the method may

- raise an exception and in this case the error message will probably be helpful
- return a [`ChatMember`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.chatmember.html#telegram.ChatMember) instance. In that case make sure to check the [`ChatMember.status`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.chatmember.html#telegram.ChatMember.status) attribute

Since API 5.1 (PTB v13.4+) you can alternatively use the [`ChatMember`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.chatmemberupdated.html) updates to keep track of users in channels. See [`chatmemberbot.py`](https://github.com/python-telegram-bot/python-telegram-bot/tree/master/examples#chatmemberbotpy) for an example.

If the user has not yet joined the channel, you can ignore incoming updates from that user or reply to them with a corresponding warning. A convenient way to do that is by using [TypeHandler](https://python-telegram-bot.readthedocs.io/en/stable/telegram.ext.typehandler.html). Read this [section](#how-do-i-limit-who-can-use-my-bot-) to learn how to do it.

## How do I send a message to all users of the bot?

Let's first point out an easy alternative solution: Instead of sending the messages directly through your bot, you can instead set up a channel to publish the announcements. You can link your users to the channel in a welcome message.

If that doesn't work for you, here we go:

To send a message to all users, you of course need the IDs of all the users. You'll have to keep track of those yourself. The most reliable way for that are the [`my_chat_member`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.chatmemberupdated.html) updates. See [`chatmemberbot.py`](https://github.com/python-telegram-bot/python-telegram-bot/tree/master/examples#chatmemberbotpy) for an example on how to use them.

If you didn't keep track of your users from the beginning, you may have a chance to get the IDs anyway, if you're using persistence. Please have a look at [this issue](https://github.com/python-telegram-bot/python-telegram-bot/issues/1836) in that case.

Even if you have all the IDs, you can't know if a user has blocked your bot in the meantime. Therefore, you should make sure to wrap your send request in a `try-except` clause checking for [`telegram.error.Unauthorized`](https://python-telegram-bot.readthedocs.io/en/stable/telegram.error.html#telegram.error.Unauthorized) errors.

Finally, note that Telegram imposes some limits that restrict you to send ~30 Messages per second. If you have a huge user base and try to notify them all at once, you will get flooding errors. To prevent that, try spreading the messages over a long time range. A simple way to achieve that is to leverage the [`JobQueue`](Extensions-–-JobQueue).