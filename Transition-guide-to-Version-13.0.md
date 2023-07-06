## Table of contents
- [Deprecations](#deprecations)
  * [Old handler API](#old-handler-api)
  * [Python 3.5](#python-35)
  * [`Message.default_quote`](#-messagedefault-quote-)
  * [`@run_async`](#run_async)
- [Persistence of Bots](#persistence-of-bots)
  * [Converting existing pickle files](#converting-existing-pickle-files)
- [API Keyword Arguments](#api-keyword-arguments)
- [JobQueue Refactored](#jobqueue-refactored)
  * [Handling of timezones](#handling-of-time-zones)
  * [Changes in advanced scheduling](#changes-in-advanced-scheduling)
    + [New features](#new-features)
    + [Changes](#changes)
  * [Setting up a `JobQueue`](#setting-up-a--jobqueue-)
- [Rich Comparison](#rich-comparison)
  * [Special note on `Message`](#special-note-on--message-)
- [Refactoring of Filters](#refactoring-of-filters)
- [Special Note on `Updater.start_webhook`](#special-note-on-updaterstart_webhook)

# Deprecations

## Old handler API

The context-based API introduced in v12 is now the default, i.e. the `use_context` argument of the `Dispatcher`/`Updater` now defaults to `True`.

## Python 3.5

As Python 3.5 reached its end of life on 2020-09-05, v13 drops support for Python 3.5. More precisely, some Python 3.6+-only features are introduced, making PTB incompatible with Python 3.5 as of v13.

## `Message.default_quote`

`Message` objects no longer have a `default_quote` attribute. Instead, `Message.bot.defaults.quote` is used. This happened in accordance with the refactoring of persistence of Bots.

## `@run_async`

It has been a long-standing issue that methods decorated with `@run_async` have not received proper error handling. We therefore decided to *deprecate* the decorator. To run functions asynchronously, you now have two options, both of which support error handling:

1. All the `Handler` classes have a new parameter `run_async`. By instantiating your handler as e.g. `CommandHandler('start', start, run_async=True)`, the callback (here `start`) will be run asynchronously.
2. To run custom functions asynchronously, you can use `Dispatcher.run_async`. Here is a small example:

    ```python
    def custom_function(a, b=None):
        pass

    def my_callback(update, context):
        a = 7
        b = 'b'
        context.dispatcher.run_async(custom_function, a, update=update, b=b)
    ```
    Of course the use of  `Dispatcher.run_async` is not limited to handler callbacks and you don't have to pass an `update` in that case. Passing the `update` when possible is just preferable because that way it's available in the error handlers.

While `@run_async` will still work, we recommend switching to the new syntax, as the decorator will be deprecated over the course of the next releases. 

# Persistence of Bots

Storing `Bot` objects (directly or e.g. as attributes of an PTB object) may lead to problems. For example, if you change
the `Defaults` object passed to your `Updater`, you would expect the loaded `Bot` instances to use the new defaults.
For that reason, v13 makes two changes:

1. `Bot` instances are no longer picklable
2. Instead, all subclasses of `BasePersistence` will replace all* `Bot` instances with a placeholder. When loading the data again,
the new `Bot` instance will be inserted.

Note that changing the used bot token may lead to e.g. `Chat not found` errors.

*Alright, almost all instances. For the limitations, see [`replace_bot`](https://python-telegram-bot.readthedocs.io/telegram.ext.basepersistence.html#telegram.ext.BasePersistence.replace_bot) and [`insert_bot`](https://python-telegram-bot.readthedocs.io/telegram.ext.basepersistence.html#telegram.ext.BasePersistence.insert_bot).

## Converting existing pickle files

In order for v13s `PicklePersistence` to be able to read your pickle files, you need to convert them once *before* upgrading to v13.* We have prepared a [Gist](https://gist.github.com/Bibo-Joshi/5fd32dde338fba474fb15f40909c92f8) for that. Use [this version](https://gist.github.com/Bibo-Joshi/23b8c02249264256ded07661d0cd8a4f), if you're upgrading directly to v13.1. This is of course only needed if you store `Bot` instances somewhere. But if you're not sure, just run the Gist ;)

If you have a custom implementation of `BasePersistence` and you currently store `Bot` instances (or any PTB object that has a `bot` attribute, e.g. `Message`), you may need to do something similar. The above Gist is a good starting point in that case.

*This is due to the fact that `PicklePersistence` uses `deepcopy`, which in turn uses the same interface as `pickle` and `Bots` are no longer pickable in v13…

# API Keyword Arguments

Pre v13, the Bot methods accepted arbitrary keyword arguments via the `**kwargs` parameter. This was implemented in order to allow for minor API updates to be patched into PTB calls while there was no release of the library just yet. However, this also lead to editors not highlighting typos. For this reason, the `**kwargs` parameter was replaced by the `api_kwargs` parameter. This means that function calls like

```python
bot.send_message(..., custom_kwarg=42)
```

need to be changed to

```python
bot.send_message(..., api_kwargs={'custom_kwarg': 42})
```
As PTB is currently up to date with the Telegram API, this shouldn't affect your code. If it does, it probably means that you had a typo somewhere ;)

# JobQueue Refactored

Previously, PTB implemented the scheduling of tasks inside the `JobQueue` manually. As timing logic is not always straightforward, maintaining the `JobQueue` was not easy and new features were only reluctantly added. To decrease development effort in that area, we refactored the `JobQueue` in v13. Now, it relies on the third party library [APScheduler](https://apscheduler.readthedocs.io/) behind the scenes.

But what does this mean for you in detail? If you're scheduling tasks vanilla style as e.g.

```python
context.job_queue.run_once(callback, when)
```

you will only have to change the handling of time zones, or likely nothing at all. In fact, everything covered in this [[wiki article|Extensions---JobQueue]] will work unchanged except for time zones. So before bothering to read on, just try to run you bot - in most cases it will still work. However, there are some more advanced things which changed.

## Handling of time zones

The `APScheduler` library only accepts time zones provided by the `pytz` library. If you pass time zone aware objects for job creating, you will need to take that into account.

## Changes in advanced scheduling

Leveraging the APScheduler library brings both some perks in terms of new features as well as some breaking changes. Please keep in mind:

* PTBs `JobQueue` provides an easy and ready to use way of scheduling tasks which ties in with the PTB architecture
* Managing scheduling logic is not the main intend of PTB and hence a third party library is used for that now
* If you need highly customized scheduling, you *can* use these advanced features of the third party library
* We can't guarantee that the back end will stay the same forever. For example, if APScheduler is discontinued, we will have to look for alternatives.

That said, here are the perks and changes:

### New features

* `run_repeating` now has a `last` parameter as originally proposed in #1333 
* `JobQueue.run_custom` allows you to run a job with a custom scheduling logic. See the APS [User Guide](https://apscheduler.readthedocs.io/userguide.html) and the page on [how to combine triggers](https://apscheduler.readthedocs.io/modules/triggers/combining.html#module-apscheduler.triggers.combining) for more details.
* All methods `JobQueue.run_*` now have a `job_kwargs` argument that accepts a dictionary. Use this to specify additional kwargs for [`JobQueue.scheduler.add_job()`](https://apscheduler.readthedocs.io/modules/schedulers/base.html#apscheduler.schedulers.base.BaseScheduler.add_job).
* Persistence of jobs: APScheduler has it's own logic of persisting jobs. Because of the aforementioned reasons, we decided to not integrate this logic with PTBs own persistence logic (at least for now). You may however set up persistence for jobs yourself. See the APS [User Guide](https://apscheduler.readthedocs.io/userguide.html) for details.

### Changes

Most importantly, the `Job` class is now a wrapper for APSchedulers own `Job` class, i.e. `job.job` is an `apscheduler.job` (don't get confused here!). In particular, attributes like `days`, `interval` and `is_monthly` were removed. Some of those could previously be used to alter the scheduling of the job. You will now have to use `job.job.modify` for that. Please see the [APScheduler docs](https://apscheduler.readthedocs.io/modules/job.html#apscheduler.job.Job.modify) for details.

There are some other minor changes, most of which will likely not affect you. For details, please see the documentation of [`JobQueue`](https://python-telegram-bot.readthedocs.io/telegram.ext.jobqueue.html) and [`Job`](https://python-telegram-bot.readthedocs.io/telegram.ext.job.html).

## Setting up a `JobQueue`

Passing a `Bot` instance to the `JobQueue` has been deprecated for a while now. v13 removes this completely. Use `JobQueue.set_dispatcher()` instead.

# Rich Comparison

v13 adds rich comparison to more Telegram objects. This means that e.g. `inline_keyboard_1 == inline_keyboard_2` is not equivalent to `inline_keyboard_1 is inline_keyboard_2`, but all the buttons will be compared. For each class supporting rich comparison, the documentation now explicitly states how equality of the class objects is determined. Warnings will be raised when trying to compare Telegram objects that don't support rich comparison.

## Special note on `Message`

Pre-v13, comparing `Message` objects only compared the `message_id`. As those are not globally unique, as of v13, `message.chat` is compared as well, i.e. messages with the same `message_id` sent in different chats are no longer evaluated as equal. While strictly speaking this is a breaking change, it shouldn't affect your code.

# Refactoring of Filters

In order to reduce confusion over the arguments of the `filter()` method, the handling of message filters vs update filters was refined. v13 brings two classes

* `telegram.ext.MessageFilter`, where `MessageFilter.filter()` accepts a `telegram.Message` object (the `update.effective_message`)

and

* `telegram.ext.UpdateFilter`, where `UpdateFilter.filter()` accepts a `telegram.Update` object (the `update`),

both inheriting from `BaseFilter`.

If you have custom filters inheriting from `BaseFilter`, you will need to change their parent class to `MessageFilter` or, if you're currently setting `update_filter = True`, to `UpdateFilter`. In that case, you can remove the `update_filter = True`.

# Special Note on `Updater.start_webhook`

If you're upgrading directly to v13.4+ and use something like

```python
updater.start_webhook(…)
updater.bot.set_webhook(my_url)
```
you will have to change that to
```python
updater.start_webhook(…, webhook_url=my_url)
```
