---

### Note: This page is a draft for the not-yet-released v5.0 of this library. See [#307](https://github.com/python-telegram-bot/python-telegram-bot/pull/307)

---

## Introduction
The extension class `telegram.ext.JobQueue` allows you to perform tasks with a delay or even periodically, at a set interval. Among many other things, you can use it to send regular updates to your subscribers.

## Usage
The `JobQueue` class is tightly integrated with other `telegram.ext` classes. Similar to `Updater` and `Dispatcher`, it runs asynchronously in a separate thread.

To use the `JobQueue`, you don't have to do much. When you instantiate the `Updater`, it will create a `JobQueue` for you:

```python
>>> from telegram.ext import Updater
>>> u = Updater('TOKEN')
>>> j = u.job_queue
```

This job queue is also linked to the dispatcher, which is discussed later in this article. Just know that unless you have a good reason to do so, you should not instantiate `JobQueue` yourself.

Tasks in the job queue is encapsulated by the `Job` class. It takes a callback function as a parameter, which will be executed when the time comes. This callback function always takes the two parameters `bot` and `job`. Similar to handler callbacks used by the `Dispatcher`, `bot` is the `telegram.Bot` instance from your `Updater`. `job` is the `Job` instance of that task (more on that later).

By default, the updater creates a job queue, but not starts it. Only when you add the first job to the queue, it will be started. To add jobs to the job queue, use the `JobQueue.put` method. You can pass `prevent_autostart=True` to this method to prevent the queue from starting automatically. 

Add your first job to the queue by defining a callback function and instantiating a `Job`. For this tutorial, you can replace `'@examplechannel'` with a channel where your bot is an admin, or by your user id (use [@userinfobot](https://telegram.me/userinfobot) to find out your user id):

```python
>>> from telegram.ext import Job
>>> def callback_minute(bot, job):
...     bot.sendMessage(chat_id='@examplechannel', text='One message every minute')
...
>>> job_minute = Job(callback_minute, 60.0, next_t=0.0)
>>> j.put(job_minute, prevent_autostart=True)
```

The `callback_minute` function will be executed every `60.0` seconds, the first time being immediately after the `JobQueue` will be started (because of `next_t=0`). 

You can also add a job that will be executed only once, with a delay:

```python
>>> def callback_30(bot, job):
...     bot.sendMessage(chat_id='@examplechannel', text='A single message with 30s delay')
...
>>> j.put(Job(callback_30, 30.0, repeat=False))
```

Because you didn't prevent the auto-start of the job queue this time, it will start processing its jobs. You should receive the message from `callback_minute` now, and in thirty seconds you should receive the message from `callback_30`. 

If you are tired of receiving a message every minute, you can temporarily disable a job or even completely remove it from the queue:

```python
>>> job_minute.enabled = False  # Temporarily disable this job
>>> job_minute.schedule_removal()  # Remove this job completely
```

**Note:** `schedule_removal` does not immediately remove the job from the queue. Instead, it is marked for removal and will be removed as soon as its current interval is over (it will not run again after being marked for removal).

A job can also change its own behavior, as it is passed to the callback function as the second argument:

```python
>>> def callback_increasing(bot, job):
...     bot.sendMessage(chat_id='@examplechannel',
...                     text='Sending messages with increasing delay up to 10s, then stops.')
...     job.interval += 1.0
...     if job.interval > 10.0:
...         job.schedule_removal()
...
>>> j.put(Job(callback_increasing, 1.0))
```

This job will send a first message after one second, a second message after two _more_ seconds, a third message after _three more_ seconds, and so on. After the ten messages, the job will terminate itself.

You might want to add jobs in response to certain user input, and there is a convenient way to do that. All `Handler` classes can pass the job queue into their callback functions, if you need them to. To do that, simply set `pass_job_queue=True` when instantiating the Handler. Another feature you can use here is the `context` keyword argument of `Job`. Let's see how it looks in code:

```python
>>> from telegram.ext import CommandHandler
>>> def callback_alarm(bot, job):
...     bot.sendMessage(chat_id=job.context, text='BEEP')
...
>>> def callback_timer(bot, update, job_queue):
...     bot.sendMessage(chat_id=update.message.chat_id,
...                     text='Setting a timer for 1 minute!')
...     
...     job_alarm = Job(callback_alarm,
...                     60.0,
...                     repeat=False,
...                     context=update.message.chat_id)
...     job_queue.put(job_alarm)
...
>>> timer_handler = CommandHandler('timer', callback_timer, pass_job_queue=True)
>>> u.dispatcher.add_handler(timer_handler)
```

By placing the `chat_id` in the `Job` object, the callback function knows where it should send the message.

All good things come to an end, so when you stop the Updater, the related job queue will be stopped as well:

```python
>>> u.stop()
```

Of course, you can instead also stop the job queue by itself:

```python
>>> j.stop()
```