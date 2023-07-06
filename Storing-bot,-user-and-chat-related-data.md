Sometimes you need to temporarily store some information about the current user and/or chat for later use. An example of this would be a survey bot that asks the user a series of questions one after another and saves them to your database when all answers are collected. 

# `bot_data`, `user_data` and `chat_data`

The `telegram.ext` framework provides a built-in solution for this common task. Let's jump straight to an example:

```python
from uuid import uuid4
from telegram.ext import Application, CommandHandler

async def put(update, context):
    """Usage: /put value"""
    # Generate ID and separate value from command
    key = str(uuid4())
    # We don't use context.args here, because the value may contain whitespaces
    value = update.message.text.partition(' ')[2]

    # Store value
    context.user_data[key] = value
    # Send the key to the user
    await update.message.reply_text(key)

async def get(update, context):
    """Usage: /get uuid"""
    # Separate ID from command
    key = context.args[0]

    # Load value and send it to the user
    value = context.user_data.get(key, 'Not found')
    await update.message.reply_text(value)

if __name__ == '__main__':
    application = Application.builder().token('TOKEN').build()

    application.add_handler(CommandHandler('put', put))
    application.add_handler(CommandHandler('get', get))
    application.run_polling()
```

## Explanation
By using `context.user_data` in any `Handler` callback, you have access to a user-specific `dict`.

*Every time the bot receives a message*, the handler for that message finds (or creates) the `user_data` of the user who sent the message. This dictionary is *shared across all handlers* of the bot.

## What about `bot_data` and `chat_data`?
`chat_data` works in the exact same way as `user_data`, except it is managed per *chat* instead of every *user*. Use `context.chat_data` to get access to this dict. As of version 12.4 `bot_data` is provided as well and works in the exact same way as `user_data`, except it's a single dictionary for your bot. Use `context.bot_data` to get access to this dict.

## Notes & Tips
- **Everything is stored in memory.** This means that all `bot_data`, `user_data` and `chat_data` is deleted when the bot process ends. If you don't want this, have a look at the [persistence page](Making-your-bot-persistent).
 - If not empty, `bot_data`, `user_data` and `chat_data` will be kept until the process ends.
- `user_data` and `chat_data` are different dictionaries even for private chats.
- You can not assign a new value to `bot_data`, `user_data` or `chat_data`. Instead of `user_data = {}` and `user_data = other_dict`, use `user_data.clear()` and/or `user_data.update(other_dict)` respectively.

## Chat Migration
If a group chat migrates to supergroup, its chat id will change. Since the `chat_data` dicts are stored *per chat id* you'll need to transfer the data to the new id. Here are the two situations you may encounter:

### Status Updates sent by Telegram
When a group migrates, Telegram will send an update that just states the new info. In order to catch those, simply define a corresponding handler:

```python
async def chat_migration(update, context):
    message = update.message
    application = context.application
    application.migrate_chat_data(message=message)
...

def main():
    application = Application.builder().token('TOKEN').build()
    application.add_handler(
        MessageHandler(filters.StatusUpdate.MIGRATE, chat_migration)
    )

...
```
See also: [`migrate_chat_data`](https://python-telegram-bot.readthedocs.io/telegram.ext.application.html#telegram.ext.Application.migrate_chat_data)

To be entirely sure that the update will be processed by this handler, either add it first or put it in its own group.

### ⚠️ Note: Migration update comes duplicated
> TLDR: Just ignore the second update

You may notice that your migration handler receives 2 updates consecutively when a group is migrated to a supergroup. The first update communicates the migration, and the second one _does the same thing_, but Telegram sends it with `from_user` set to an Anonymous user `GroupAnonymousBot`. They do this so older clients of Telegram - where every `Update` needs a `from_user` - don't crash. You can simply ignore the second update as it brings the same information but with different fields :)

### ChatMigrated Errors

If you try e.g. sending a message to the old chat id, Telegram will respond by a `BadRequest` including the new chat id. You can access it using an error handler:

```python
async def error(update, context):
    """Log Errors caused by Updates."""
    logger.warning('Update "%s" caused error "%s"', update, context.error)

    if isinstance(context.error, ChatMigrated):
        new_chat_id = context.error.new_chat_id
```
Unfortunately, Telegram does *not* pass along the old chat id, so there is currently no simple way to perform a data transfer like above within the error handler. So make sure, that you catch the status updates! Still, you can wrap your requests into a `try-except`-clause:

```python
async def my_callback(update, context):
    application = context.application
    ...

    try:
        await context.bot.send_message(chat_id, text)
    except ChatMigrated as exc:
        new_id = exc.new_chat_id

        # Resend to new chat id
        await context.bot.send_message(new_id, text)

        # Get old and new chat ids
        old_id = update.message.migrate_from_chat_id or message.chat_id
        new_id = update.message.migrate_to_chat_id or message.chat_id

        # transfer data, only if old data is still present
        # this step is important, as Telegram sends *two* updates
        # about the migration
        if old_id in application.chat_data:
        application.migrate_chat_data(
            old_chat_id=old_id,
            new_chat_id=new_id
        )
    ...
```
