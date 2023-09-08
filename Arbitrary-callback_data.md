# Arbitrary objects as `InlineKeyboardButton.callback_data`

The Telegrams Bot API only accepts strings with length up to 64 bytes as `callback_data` for `InlineKeyboardButtons`, which sometimes is quite a limitation.

With PTB, you are able to pass *any* object as `callback_data`. This is achieved by storing the object in a cache and passing a unique identifier for that object to Telegram. When a `CallbackQuery` is received, the id in the `callback_data` is replaced with the stored object. To use this feature, set [`Application.arbitrary_callback_data`](https://docs.python-telegram-bot.org/telegram.ext.applicationbuilder.html#telegram.ext.ApplicationBuilder.arbitrary_callback_data) to `True`. The cache that holds the stored data has limited size (more details on memory usage below). If the cache is full and objects from a new `InlineKeyboardMarkup` need to be stored, it will discard the data for the least recently used keyboard.

This means two things for you:

1. If you don't use [persistence](../wiki/Making-your-bot-persistent), buttons won't work after restarting your bot, as the stored updates are lost. More precisely, the `callback_data` you will receive is an instance of  `telegram.ext.InvalidCallbackData`. If you don't need persistence otherwise, you can set `store_callback_data` to `True` and all the others to `False`.
2. If you have a number of keyboards that need to stay valid for a very long time, you might need to do some tweaking manually (see below)
3. When using the `CallbackQueryHandler`, the `pattern` argument can now be either

    * a regex expression, which will be used, if the `callback_data` is in fact a string
    * a callable accepting the `callback_data` as only argument. You can perform any kinds of tests on the `callback_data` and return `True` or `False` accordingly
    * a type. In that case the `CallbackQuery` will be handled, if the `callback_data` is an instance of that type. Btw: This allows you to inform users, when a buttons' data has been dropped from cache. With `CallbackQueryHandler(callback, pattern=InvalidCallbackData)` you can e.g., call `await update.callback_query.answer(text='Button is no longer valid', show_alert=True)` to inform the user.

## Memory Usage

PTB stores the callback data objects in memory. Additionally, to that, it stores a mapping of `CallbackQuery.id` to the corresponding UUID. By default, both storages contain a maximum number of 1024 items. You can change the size by passing an integer to the `arbitrary_callback_data` argument of `ApplicationBuilder`/`ExtBot`.

As PTB can't know when the stored data is no longer needed, it uses an LRU (Least Recently Used) cache. This means that when the cache is full, it will drop the keyboard that has been not used for the longest time. However, if you want to keep memory usage low, you have 
additional options to drop data:

* on receiving a `CallbackQuery`, you can call `context.drop_callback_data(callback_query)`. This will delete the data associated with the keyboard attached to the message that originated the `CallbackQuery`. Calling `context.drop_callback_data` is safe in any case where you change the keyboard, i.e. `callback_query.edit_message_text/reply_markup/…`
  **Note:** If the user clicks a button more than one time fast enough, but you call `context.drop_callback_data` for the first resulting `CallbackQuery`, the second one will have `InvalidCallbackData` as `callback_data`. However, this is usually not a problem, because one only wants one button click anyway.
* To drop more data from memory, you can call `bot.callback_data.clear_callback_queries()` or `bot.callback_data.clear_callback_data()`, which will drop the mapping of `CallbackQuery` ids to the associated UUID and the mapping of UUIDs to data, respectively. `clear_callback_data` also accepts a `time_cutoff`, allowing you to delete only entries older than a specified time.

## Security of InlineKeyboardButtons

Callback updates are not sent by Telegram, but by the client. This means that they can be manipulated by a user. (While Telegram unofficially does try to prevent this, they don't want Bot devs to rely on them doing the security checks).

Most of the time, this is not really a problem, since `callback_data` often just is `Yes`, `No`, etc. However, if the callback data is something like `delete message_id 123`, the malicious user could delete any message sent by the bot.

When using `arbitrary_callback_data` as described above, PTB replaces the outgoing `callback_data` with a [UUID](https://docs.python.org/3/library/uuid.html), i.e., a random unique identifier. This makes the `callback_data` safe: If a malicious client alters the scent `CallbackQuery`, the invalid UUID can't be resolved. In this case `CallbackQuery.data` will be an instance of `telegram.ext.InvalidCallbackData`. Note that this is also the case, when the UUID *was* valid, but the data has already been dropped from cache - PTB can't distinguish between the two cases.

## Manually handling updates

You may be manually building your updates from JSON-data, e.g., in case you are using a custom webhooks setup. In this case you'll have to make sure that the cached data is inserted back into the buttons yourself, like this:

```python
update = Update.de_json(data, bot)
if bot.arbitrary_callback_data:
    bot.insert_callback_data(update)
```

## Special note about channel posts

Inline buttons are not only sent, your bot also receives them. In the return value of your bot message, when receiving messages that are replies to messages with an inline keyboard, have `message.pinned_message` or where `message.via_bot` is your bot (i.e., messages sent via your bot in inline mode). PTB tries very hard to insert the corresponding data back into all those keyboards, where appropriate - i.e., where the keyboard was sent by __your__ bot and not by another bot. There is however one case, where there is no way to tell that: channel posts have no `from_user`. So unless they have the `via_bot` attribute, there is no way to tell, if they were sent by your bot or another one. This means:

If your bot receives a channel post, which as `reply_to_message` or `pinned_message` and the latter has a keyboard, but was __not__ sent by your bot, all `callback_data` will contain `InvalidCallbackData` instances. This is of course unfortunate, but we do have a feeling that the cases where this would complicate things are so rare that it doesn't really matter 😉 