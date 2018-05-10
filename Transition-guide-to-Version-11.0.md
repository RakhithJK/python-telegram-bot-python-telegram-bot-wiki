## Table of contents
* [Context based callbacks](#context-based-callbacks)
    * [Handler callbacks](#handler-callbacks)
    * [Error handler callbacks](#error-handler-callbacks)
    * [Job callbacks](#job-callbacks)
    * [What exactly is `CallbackContext`](#what-exactly-is-callbackcontext)
    * [Note about group and groupdict](#note-about-group-and-groupdict)
    * [Note about version 12](#note-about-version-12)
    * [Custom handlers](#custom-handlers)
* [Filters in handlers](#filters-in-handlers)

# Context based callbacks
The biggest change in this release is context based callbacks. When running your bot you will probably see a warning like the following:
```
echobot2.py:62: TelegramDeprecationWarning: Old Handler API is deprecated - see https://git.io/vp113 for details
```
This means you're using the old style callbacks, and should upgrade to context based callbacks.

The first thing you should do is find where you create your `Updater`.
``` python
updater = Updater('TOKEN')
```
And add `use_context=True` so it looks like
```python
updater = Updater('TOKEN', use_context=True)
```
**Note that this is only necessary in version 11 of `python-telegram-bot`. Version 12 will have `use_context=True` set as default.**
_If you do **not** use `Updater` but only `Dispatcher` you should instead set `use_context=True` when you create the `Dispatcher`._

## Handler callbacks
Now on to the bulk of the change. You wanna change all your callback functions from the following:
``` python
def start(bot, update, args, job_queue):
    # Stuff here
```

to the new style using CallbackContext
``` python
def start(update: Update, context: CallbackContext):
    # Stuff here
    # args will be available as context.args
    # jobqueue will be available as context.jobqueue
```
_On python 2 which doesn't support annotations replace `update: Update, context, CallbackContext` with simply `update, context`._

## Error handler callbacks
Error handler callbacks are the ones added using `Dispatcher.add_error_handler`. These have also been changed from the old form:
```python
def error_callback(bot, update, error):
    logger.warning('Update "%s" caused error "%s"', update, error)
```
into
```python
def error_callback(update, context):
    logger.warning('Update "%s" caused error "%s"', update, context.error)
```
_Note that the error is now part of the `CallbackContext` object._

## Job callbacks
Job callbacks (the ones that are passed to `JobQueue.run_once` and similar) have also changed. Old form:
``` python
def job_callback(bot, job):
   bot.send_message(SOMEONE, job.context)
```
New form:
``` python
def job_callback(context):
    job = context.job
    context.bot.send_message(SOMEONE, job.context)
```
_Note that both bot, and job have been merged into the `CallbackContext` object._

## What exactly is `CallbackContext`
`CallbackContext` is an object that contains all the extra context information regarding an update. It replaces the old behaviour with having a ton of `pass_something=True` in your handlers. Instead, all this data is availible directly on the `CallbackContext` - always!

## Note about groups and groupdict
Before version 11, you could both pass_groups and pass_groupdict. Inside `CallbackContext` this has been combined into a single `Match` object. Therefore if your handler looked like this before:
``` python
def like_callback(bot, update, groups, groupdict): # Registered with a RegexHandler with pattern (?i)i (like|dislike) (?P<thing>.*)
    update.reply_text('You {} {}'.format(groups[1], groupdict['thing'])
```
It would instead now look something like this:
``` python
def like_callback(update, context): # Registered with a RegexHandler with pattern (?i)i (like|dislike) (?P<thing>.*)
    update.reply_text('You {} {}'.format(context.match[1], context.match.groupdict()['thing'])
```

## Note about version 12
In version 12 of `python-telegram-bot`, `use_context` will default to `True`. This means that your old handlers using pass_ will stop working. It also means that after upgrading to version 12, you can remove `use_context=True` from your `` if you so desire.

# Custom handlers
This part is only relavant if you've developed custom handlers, that subclass `telegram.ext.Handler`. To support the new context based callbacks, add a method called `collect_additional_context` to your handler. The method receives a `CallbackContext` object, and should add whatever extra context is needed (at least everything that could be added via `pass_` arguments before). Note that `job_queue, update_queue, chat_data, user_data` is automatically added by the base `Handler`.

***

# Filters in handlers
Using a list of filters in a handler like below has been deprecated for a while now. Version 11 removes the ability completely.
``` python
MessageHandler([Filters.audio, Filters.video], your_callback)
```
## Combine filters using bitwise operators
Instead you can now combine filters using bitwise operators like below. (The pipe ( `|` ) character means OR, so the below is equalivant to the above example using a list).
``` python
# Handle messages that contain EITHER audio or video
MessageHandler(Filters.audio | Filters.video, your_callback)
```
### Also supports AND, NOT and more than 3 filters
**AND:**
```python
# Handle messages that are text AND contain a mention
Filters.text & Filters.entity(MesageEntity.MENTION)
```
**NOT:**
``` python
# Handle messages that are NOT commands (same as Filters.text in most cases)
~ Filters.command
```
**More advanced combinations:**
``` python
# Handle messages that are text and contain a link of some kind
Filters.text & (Filters.entity(URL) | Filters.entity(TEXT_LINK))
# Handle messages that are text but are not forwarded
Filters.text & (~ Filters.forwarded)
```