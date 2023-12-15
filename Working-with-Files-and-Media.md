Bots interacting with users in plain text messages is often times not enough for a pleasant user experience.
Providing the users with images, videos, files and other media is therefore a common use case for bot programmers and the Bot API provides several ways to do this.
On this wiki page, we explain how files and media are handled in the `python-telegram-bot` framework.

- [Sending files](#sending-files)
  - [Sending a media group](#sending-a-media-group)
  - [Sending files via inline mode](#sending-files-via-inline-mode)
- [Editing a file](#editing-a-file)
- [Downloading a file](#downloading-a-file)

## Sending files

If you want to send a file (e.g. send a document or a photo) with the bot, you have three options:

* Upload the file
* Send an HTTP URL that leads to the file
* Send a `file_id` of a file that has already been sent.

Note that not every method is supported everywhere (e.g. for thumbnails you can't pass a `file_id`). Make sure to check out the documentation of the corresponding bot method for details.

Please also check out the [official Telegram API docs](https://core.telegram.org/bots/api#sending-files) on sending files.

Let's have a look at how sending a document can be done. In these examples, we'll be using `Bot`'s [`send_document()`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.send_document) method.

> [!NOTE]
> In discussion and examples below, we will be using methods of `Bot`, but most of them
> (including [`send_document()`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.send_document)) 
> have shortcut methods in classes like `User`, `Chat` or `Message` that can be more 
> convenient to use in your particular situation. Documentation for every method in `Bot`
> contains links to shortcut methods in other classes.


1. Uploading a file

    ```python
    await bot.send_document(chat_id=chat_id, document=open('tests/test.png', 'rb'))
    ```
    or even just 

    ```python
    await bot.send_document(chat_id=chat_id, document='tests/test.png')
    ```
    When you pass a file path (note that both `str` and [`pathlib.Path`](https://docs.python.org/3/library/pathlib.html#pathlib.Path) are accepted as [`document`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.send_document.params.document) parameter), PTB will automatically check if your bot is running in [local mode](https://github.com/python-telegram-bot/python-telegram-bot/wiki/Local-Bot-API-Server#how-to-use-a-local-bot-api-server-with-ptb). If it is, the file does not need to be uploaded. Otherwise, the file is read in binary mode, so just as when you pass `open('tests/test.png', 'rb')`.

2. Sending an HTTP URL

    ```python
    await bot.send_document(chat_id=chat_id, document='https://python-telegram-bot.org/static/testfiles/telegram.gif')
    ```

3. Sending by `file_id`:

    ```python
    await bot.send_document(chat_id=chat_id, document=file_id)
    ```

    Two further notes on this:
    
    1. Each bot has its own `file_id`s, i.e. you can't use a `file_id` from a different bot to send a photo
    2. How do you get a `file_id` of a photo you sent? Read it from the return value of [`bot.send_document()`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.send_document) (or any other [`Message`](https://python-telegram-bot.readthedocs.io/telegram.message.html#telegram.Message) object you get your hands on):
    
        ```python
        message = await bot.send_document(...)
        file_id = message.document.file_id
        ```
       
This pretty much works the same way for all the other `send_<media_type>()` methods like [`send_photo()`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.send_photo), [`send_video()`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.send_video) etc. There is one exception, though: [`send_media_group()`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.send_media_group). 

### Sending a media group

A call to [`send_media_group()`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.send_media_group) looks like this:

```python
await bot.send_media_group(chat_id=chat_id, media=[media_1, media_2, ...])
```

Each of the items in the `media` sequence (list or tuple) must be an instances of [`InputMediaAudio`](https://python-telegram-bot.readthedocs.io/telegram.inputmediaaudio.html#telegram-inputmediaaudio), [`InputMediaDocument`](https://python-telegram-bot.readthedocs.io/telegram.inputmediadocument.html#telegram-inputmediadocument), [`InputMediaPhoto`](https://python-telegram-bot.readthedocs.io/telegram.inputmediaphoto.html#telegram-inputmediaphoto) or [`InputMediaVideo`](https://python-telegram-bot.readthedocs.io/telegram.inputmediavideo.html#telegram-inputmediavideo). The media comes into play like so:

```python
media_1 = InputMediaDocument(media=open('tests/test.png', 'rb'), ...)
media_1 = InputMediaDocument(media='https://python-telegram-bot.org/static/testfiles/telegram.gif', ...)
media_1 = InputMediaDocument(media=file_id, ...)
```

> [!CAUTION]
> For the `InputMedia*` classes, passing a file path only works if your bot is running in [local mode](docs.python-telegram-bot.org/telegram.bot.html#telegram.Bot.params.local_mode).

### Sending files via inline mode

You may want to allow users to send media via your bots inline mode. This works a little bit different than posting media via `send_*`. Most notably, you can't upload files for inline mode! You must provide either an HTTP URL or a `file_id`.

Let's stick to example of sending a document. You have to provide [`bot.answer_inline_query()`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.answer_inline_query) with an [`InlineQueryResult`](https://python-telegram-bot.readthedocs.io/telegram.inlinequeryresult.html#telegram-inlinequeryresult) that represents that document. There are two ways of doing that:

1. HTTP URL:

    ```python
    result = InlineQueryResultDocument(document_url='https://python-telegram-bot.org/static/testfiles/telegram.gif', ...)
    ```
   
2. `file_id`:

    ```python
    result = InlineQueryResultCachedDocument(document_file_id=file_id, ...)
    ```

In this example, we are using [`InlineQueryResultDocument`](https://python-telegram-bot.readthedocs.io/telegram.inlinequeryresultdocument.html#telegram-inlinequeryresultdocument) for option #1 and [`InlineQueryResultCachedDocument`](https://python-telegram-bot.readthedocs.io/telegram.inlinequeryresultcacheddocument.html#telegram-inlinequeryresultcacheddocument) for option #2. The scheme `InlineQueryResult<media_type>` vs `InlineQueryResultCached<media_type>` is similar for the other media types.
Again, please check out the docs for details on required and optional arguments. 

## Editing a file

When you have sent a file, you may want to edit it. This works similarly as `send_media_group`, i.e. the media must be wrapped into a `InputMedia<media_type>` object. Again, with `document` as example, we'll call [`bot.edit_message_media()`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.edit_message_media) and pass an instance of [`InputMediaDocument`](https://python-telegram-bot.readthedocs.io/telegram.inputmediadocument.html#telegram-inputmediadocument) as `media`:

```python
await bot.edit_message_media(chat_id=chat_id, message_id=message_id, media=InputMediaDocument(media=open('tests/test.png'), ...))
```

Please check out the restrictions on editing media in the official docs of [`editMessageMedia`](https://core.telegram.org/bots/api#editmessagemedia).

## Downloading a file

When you receive files from a user, you sometimes want to download and save them. If it's a document, that could look like this:

```python
file_id = message.document.file_id
new_file = await bot.get_file(file_id)
await new_file.download_to_drive()
```

For a received video/voice/... change [`message.document`](https://python-telegram-bot.readthedocs.io/telegram.message.html#telegram.Message.document) to `message.video/voice/...`. However, there is one exception: [`message.photo`](https://python-telegram-bot.readthedocs.io/telegram.message.html#telegram.Message.photo) is a *list* of [`PhotoSize`](https://python-telegram-bot.readthedocs.io/telegram.photosize.html) objects, which represent different sizes of the same photo. Use `message.photo[-1].file_id` to get the largest size.

> **See also:** 
>
> Documentation for [`Bot.get_file()`](https://python-telegram-bot.readthedocs.io/telegram.bot.html#telegram.Bot.get_file)

Moreover, the above snippet can be shortened by using PTB's built-in utility shortcuts:

```python
new_file = await message.effective_attachment.get_file()
await new_file.download_to_drive('file_name')
```

[`message.effective_attachment`](https://python-telegram-bot.readthedocs.io/telegram.message.html#telegram.Message.effective_attachment) automatically contains whichever media attachment the message has. In case of a photo, you'll again have to use e.g. `message.effective_attachment[-1].get_file()`.

> **See also:** 
>
> Documentation for [`File.download_to_drive()`](https://python-telegram-bot.readthedocs.io/telegram.file.html#telegram.File.download_to_drive) and [`File.download_to_memory()`](https://python-telegram-bot.readthedocs.io/telegram.file.html#telegram.File.download_to_memory)

