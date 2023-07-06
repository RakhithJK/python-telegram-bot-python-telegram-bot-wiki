# Introduction
Hey, this wiki page will walk you through the inline keyboard example found [here](../blob/master/examples/inlinekeyboard.py). We will start with how python starts with the example, then follow through the code in the same way we expect updates from the user would go through it. Let's do it.

_Disclaimer: We will conveniently ignore the imports._
## Startup

```python
if __name__ == '__main__':
    main()
```
[Lines 78 to 79](../blob/master/examples/inlinekeyboard.py#L78-L79) tell python that after starting the script, it's supposed to call the main function
## main

```python
application = Application.builder().token("TOKEN").build()
```
[The first line](../blob/master/examples/inlinekeyboard.py#L68) in the main function builds an application instance from the [Application class](https://docs.python-telegram-bot.org/telegram.ext.application.html). The function calls lined up after another means that the calls happen on the return of the previous call. So `.builder()` is called on `Application`, `.token("TOKEN")` on the return of `.builder()`, `.build()` on whatever `.token("TOKEN")` returns. If you check the docs, you will find that [builder](https://docs.python-telegram-bot.org/telegram.ext.application.html#telegram.ext.Application.builder) returns an [ApplicationBuilder instance](https://docs.python-telegram-bot.org/en/stable/telegram.ext.applicationbuilder.html). So looking there for [token](https://docs.python-telegram-bot.org/telegram.ext.applicationbuilder.html#telegram.ext.ApplicationBuilder.token), we find that it also returns an (updated) `ApplicationBuilder` instance, which is the same for almost every method on that page. This allows this chaining of function calls, since all the function are defined inside the `ApplicationBuilder` and it is always returned. Finally, the `.build()` builds and then returns the application instance we expect.

```python
application.add_handler(CommandHandler("start", start))
application.add_handler(CallbackQueryHandler(button))
application.add_handler(CommandHandler("help", help_command))
```
[Line 70 to 72](../blob/master/examples/inlinekeyboard.py#L70-L72) registers our three handlers. The first handler is a [CommandHandler](https://python-telegram-bot.readthedocs.io/telegram.ext.commandhandler.html). Whenever a user sends a /start command to the bot, the function `start` is called. Same situation with the third handler: Whenever a user sends the /help command, `help_command` gets called.

The second handler is a [CallbackQueryHandler](https://docs.python-telegram-bot.org/telegram.ext.callbackqueryhandler.html). A [Callbackquery](https://docs.python-telegram-bot.org/telegram.callbackquery.html) is what Telegram sends to our bot when a user presses an [InlineButton](https://docs.python-telegram-bot.org/telegram.inlinekeyboardbutton.html). Every press of a button gets sent to the `button` handler.

```python
application.run_polling()
```
[Line 75](../blob/master/examples/inlinekeyboard.py#L75) tells the PTB library to start the bot using polling, which means that the library will continuously make a request to the telegram servers and get new updates from there, if they exists.


Let's start our way through the handlers in the same way we would expect a user to go through it. This means we begin with the start handler:
## start

```python
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
```
In [line 34](../blob/master/examples/inlinekeyboard.py#L34) we define a function called start. It is an async function and it takes the two arguments update (instance of an [Update](https://docs.python-telegram-bot.org/telegram.update.html)) and context (instance of a [CallbackContext](https://docs.python-telegram-bot.org/telegram.ext.callbackcontext.html)). The context is the default context type, since we didn't change anything with it. If you want to see how that works, checkout the [ContextType bot example](../blob/master/examples/contexttypesbot.py). The `-> None` indicates to a type checker that this function returns nothing.

```python
keyboard = [
    [
        InlineKeyboardButton("Option 1", callback_data='1'),
        InlineKeyboardButton("Option 2", callback_data='2'),
    ],
    [InlineKeyboardButton("Option 3", callback_data='3')],
]

```
[Line 36 to 42](../blob/master/examples/inlinekeyboard.py#L36-L42) a variable called keyboard is defined. It is a list of lists, representing a 2D-Matrix. Every "parent" list is a row in the actual inline keyboard (so `[[1], [2]]` would be two rows), every entry inside an parent list is a column. So this keyboard will have two rows, Option 1 and Option 2 will be in the first; Option 3 in the second one.

```python
reply_markup = InlineKeyboardMarkup(keyboard)
```
[Line 44](../blob/master/examples/inlinekeyboard.py#L44) turns our list into an actual Inline Keyboard that we can pass along with our message.

```python
await update.message.reply_text('Please choose:', reply_markup=reply_markup)
```
In [line 46](../blob/master/examples/inlinekeyboard.py#L46) we reply to the update message with a text (hence [reply_text](https://docs.python-telegram-bot.org/telegram.message.html#telegram.Message.reply_text)) and pass the keyboard along in the `reply_markup` argument. The `await` tells the program to stop and wait for the function call to finish. `async` and `await` are both fundamentals of asyncio programming in python, explaining this further is outside of this example explainer. If you are curious, feel free to search for it, otherwise just accept these keywords as they are.

Now we expect people to press one of the provided buttons, so let's jump to the button callback
## button

```python
async def button(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
```
[Line 49](../blob/master/examples/inlinekeyboard.py#L49) defines a function called button. It takes the two arguments update and context and returns nothing. Basically the same as `start`.

```python
query = update.callback_query
```
[Line 51](../blob/master/examples/inlinekeyboard.py#L51) defines query as a shortcut to access the provided [CallbackQuery](https://docs.python-telegram-bot.org/telegram.callbackquery.html). This is the part of the update which has all the information in it, remember, it gets generated/send to the bot once a user presses a button.


```python
await query.answer()
```
[Line 55](../blob/master/examples/inlinekeyboard.py#L55) here we answer the `CallbackQuery`. We use a convenient shortcut PTB provides. It takes care of calling the [actual function](https://docs.python-telegram-bot.org/telegram.bot.html#telegram.Bot.answer_callback_query) and passing all the required parameters to it. If you check out the function, you see that you can pass a `text` argument to it, which will be displayed in a little pop-up on the client end, and if you pass `show_alert` on top of it, the user has to dismiss the pop-up. Not useful for this example, so we just pass it without these optional arguments.

```python
await query.edit_message_text(text=f"Selected option: {query.data}")
```
[Line 57](../blob/master/examples/inlinekeyboard.py#L57) edits the message where `CallbackQuery` originates from with the text where we tell the user which option we picked. We insert `query.data` into the string, which is the data we defined in the keyboard, so the number 1, 2 or 3. Since we don't pass the inline keyboard along again, it will disappear.
## help

```python
async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text("Use /start to test this bot.")
```
[Line 60 to 62](../blob/master/examples/inlinekeyboard.py#L60-L62) is a simple callback. Here we reply to the /help command with the provided text: They should use /start to use this bot.
## error

```python
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO
)
logger = logging.getLogger(__name__)
```
[Line 28 to 31](../blob/master/examples/inlinekeyboard.py#L28-L31) are the only lines of the code we haven't covered yet. Here we set up the logging module to have the format we want, and we define logger in case we want to use it later. More docs regarding logging can be found [here](https://docs.python.org/3/library/logging.html)

***

This section of the wiki is currently in development, feedback is greatly appreciated. Ping one of the admins in our telegram group to anything you want to tell us.