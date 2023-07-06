# Introduction
The extension class [`telegram.ext.JobQueue`](https://docs.python-telegram-bot.org/telegram.ext.jobqueue.html#telegram.ext.JobQueue) allows you to perform tasks with a delay or even periodically, at a set interval. Among many other things, you can use it to send regular updates to your subscribers.

## When working with `JobQueue`, please keep in mind:

* PTBs `JobQueue` provides an easy to use and ready to use way of scheduling tasks in a way that ties in with the PTB architecture
* Managing scheduling logic is not the main intend of PTB and hence as of v13 a third party library is used
* If you need highly customized scheduling thingies, you *can* use advanced features of the third party library
* We can't guarantee that the backend will stay the same forever. For example, if the third party library is discontinued, we will have to look for alternatives.

## Example

In addition to the tutorial below, there is also the `timerbot.py` example at the [examples directory](https://github.com/python-telegram-bot/python-telegram-bot/tree/master/examples).

# Usage
> :warning: Since v20, you must install PTB with the optional requirement `job-queue`, i.e. 
>
> `pip install 'python-telegram-bot[job-queue]'`

The `JobQueue` class is tightly integrated with other `telegram.ext` classes.

To use the `JobQueue`, you don't have to do much. When you build the `Application`, it will create a `JobQueue` for you:

```python
from telegram.ext import Application

application = Application.builder().token('TOKEN').build()
job_queue = application.job_queue
```

Just know that unless you have a good reason to do so, you should not instantiate `JobQueue` yourself.

Tasks in the job queue are encapsulated by the [`Job`](https://python-telegram-bot.readthedocs.io/telegram.ext.job.html#telegram-ext-job) class. It takes a [callback function as a parameter](https://python-telegram-bot.readthedocs.io/telegram.ext.job.html#telegram.ext.Job.params.callback), which will be executed when the time comes. This callback function always takes exactly one parameter: `context`, a [`telegram.ext.CallbackContext`](https://python-telegram-bot.readthedocs.io/telegram.ext.callbackcontext.html). Like in the case of handler callbacks used by the `Application`, through this object you can access 
* `context.bot`, the `Application`'s `telegram.Bot` instance
* `context.job_queue`, the same object as `application.job_queue` above
* and for this particular case you can also access `context.job`, which is the `Job` instance of the task that triggered the callback (more on that later). 

You can use the following methods to create jobs with different frequency and time: `job_queue.run_once`, `job_queue.run_repeating`, `job_queue.run_daily` and `job_queue.run_monthly`. (As before, you do not usually need to instantiate the `Job` class directly.)

## Tutorial

Add your first job to the queue by defining a callback function and adding it to the job queue. For this tutorial, you can replace `'@examplechannel'` with a channel where your bot is an admin, or by your user id (use [@userinfobot](https://telegram.me/userinfobot) to find out your user id):

```python
from telegram.ext import ContextTypes, Application

async def callback_minute(context: ContextTypes.DEFAULT_TYPE):
    await context.bot.send_message(chat_id='@examplechannel', text='One message every minute')

application = Application.builder().token('TOKEN').build()
job_queue = application.job_queue

job_minute = job_queue.run_repeating(callback_minute, interval=60, first=10)

application.run_polling()
```

The `callback_minute` function will be executed every `60.0` seconds, the first time being after 10 seconds (because of `first=10`). The `interval` and `first` parameters are in seconds if they are `int` or `float`. They can also be `datetime` objects. See the [docs](http://python-telegram-bot.readthedocs.io/telegram.ext.jobqueue.html) for detailed explanation.
The return value of these functions are the `Job` objects being created. You don't need to store the result of `run_repeating` (which is the newly instantiated `Job`) if you don't need it; we will make use of it later in this tutorial.

You can also add a job that will be executed only once, with a delay:

```python
from telegram.ext import ContextTypes, Application

async def callback_30(context: ContextTypes.DEFAULT_TYPE):
    await context.bot.send_message(chat_id='@examplechannel', text='A single message with 30s delay')

application = Application.builder().token('TOKEN').build()
job_queue = application.job_queue

job_queue.run_once(callback_30, 30)

application.run_polling()
```

In thirty seconds, you should receive the message from `callback_30`. 

If you are tired of receiving a message every minute, you can temporarily disable a job or even completely remove it from the queue:

```python
job_minute.enabled = False  # Temporarily disable this job
job_minute.schedule_removal()  # Remove this job completely
```

**Note:** [`schedule_removal`](https://python-telegram-bot.readthedocs.io/telegram.ext.job.html#telegram.ext.Job.schedule_removal) does not immediately remove the job from the queue. Instead, it is marked for removal and will be removed as soon as its current interval is over (it will not run again after being marked for removal).

You might want to add jobs in response to certain user input, and there is a convenient way to do that. The `context` argument of your `Handler` callbacks has the `JobQueue` attached as `context.job_queue` ready to be used. Another feature you can use here are the [`data`](https://python-telegram-bot.readthedocs.io/telegram.ext.job.html#telegram.ext.Job.params.data), [`chat_id`](https://python-telegram-bot.readthedocs.io/telegram.ext.job.html#telegram.ext.Job.params.chat_id) or [`user_id`](https://python-telegram-bot.readthedocs.io/telegram.ext.job.html#telegram.ext.Job.params.user_id) keyword arguments of `Job`. You can pass any object as a `data` parameter when you launch a `Job` and retrieve it at a later stage as long as the `Job` exists. The `chat_id`/`user_id` parameter allows for an easy way to let the `Job` know which chat we're talking about. This way, we can access `context.chat_data`/`context.user_data` in the job's `callback`. Let's see how it looks in code:

```python
from telegram import Update
from telegram.ext import CommandHandler, Application, ContextTypes
 
async def callback_alarm(context: ContextTypes.DEFAULT_TYPE):
    # Beep the person who called this alarm:
    await context.bot.send_message(chat_id=context.job.chat_id, text=f'BEEP {context.job.data}!')
 
 
async def callback_timer(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.message.chat_id
    name = update.effective_chat.full_name
    await context.bot.send_message(chat_id=chat_id, text='Setting a timer for 1 minute!')
    # Set the alarm:
    context.job_queue.run_once(callback_alarm, 60, data=name, chat_id=chat_id)
 
application = Application.builder().token('TOKEN').build()
timer_handler = CommandHandler('timer', callback_timer)
application.add_handler(timer_handler)
application.run_polling()
```

By placing the `chat_id` in the `Job` object, the callback function knows where it should send the message.


All good things must come to an end, so when you stop the Application, the related job queue will be stopped as well.

## Persistent Job Queues

PTBs [[Persistence Setup|Making-your-bot-persistent]] currently does not support serialization of jobs.
However, the current backend of the `JobQueue`, namely the `APScheduler` library has a mechanism for that, which you can leverage.
Check out e.g. [ptbcontrib/ptb_jobstores](https://github.com/python-telegram-bot/ptbcontrib/tree/main/ptbcontrib/ptb_jobstores) for an example implementation.
