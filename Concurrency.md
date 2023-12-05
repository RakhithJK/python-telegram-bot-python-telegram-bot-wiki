# Concurrency in PTB

## Table of contents

- [Default behavior](#default-behavior)
- [Using concurrency](#using-concurrency)
  - [`Handler.block`](#handlerblock)
  - [`Application.concurrent_updates`](#applicationconcurrent_updates)
  - [`Application.create_task`](#applicationcreate_task)
- [Tailor-made Concurrency](#tailor-made-concurrency)

> ⚠️ Please make sure to read this page in its entirety and in particular the section on [tailor-made concurrency](#tailor-made-concurrency)

PTB is build on top of Pythons [`asyncio`](https://docs.python.org/3/library/asyncio.html), which allows writing concurrent code using the `async`/`await` syntax.
This greatly helps to design code that efficiently uses the wait time during I/O operations like network communication (e.g. sending a message to a user) or reading/writing on the hard drive.

**Note:**
`asyncio` code is usually single-threaded and hence PTB currently does not aim to be thread safe (see the readme for more info.)

## Default behavior

By default, incoming updates and handler callbacks are processed sequentially, i.e. one after the other.
So, if one callback function takes some time to execute, all other updates have to wait for it.

**Example:**
You're running the [Echobot](https://github.com/python-telegram-bot/python-telegram-bot/blob/master/examples/echobot.py) and two users (*User A* and *User B*) send a message to the bot at the same time.
Maybe *User A* was a bit quicker, so their request arrives first, in the form of an `Update` object (*Update A*).
The `Application` checks the `Update` and decides it should be handled by the handler with the callback function named `echo`.
At the same time, the `Update` of *User B* arrives (*Update B*).
But the `Application` is not finished with *Update A*.
It calls the `echo` function with *Update A*, which sends a reply to *User A*. Sending a reply takes some time, and *Update B* remains untouched during that time.
Only after the `echo` function finishes for *Update A*, the `Application` repeats the same process for *Update B*.

If you have handlers in multiple groups, it gets a tiny bit more complicated.
The following pseudocode explains how `Application.process_update` roughly works in the default case, i.e. sequential processing (we simplified a bit by e.g. skipping some arguments of the involved methods):

```python
async def process_update(self, update):
    # self is the `Application` instance
    for group_number, handlers in self.handlers.items():
        # `handlers` is the list of handlers in group `group_number`
        for handler in handlers:
            if handler.check_update(update):
                # Here we `await`, i.e. we only continue after the callback is done!
                await handler.handle_update(update)
                break  # at most one handler per group handles the update
```

## Using concurrency

We want to reply to both *User A* and *User B* as fast as possible and while sending the reply to user *User A* we'd like to already get started on handling *Update B*.
PTB comes with three built-in mechanism that can help with that.

### `Handler.block`

Via the `block` parameter of `Handler` you can specify that `Application.process_update` should not wait for the callback to finish:

```python
application.add_handler(
  MessageHandler(filters.TEXT & ~filters.COMMAND, echo, block=False)
)
```

Instead, it will run the callback as [`asyncio.Task`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task) via [`asyncio.create_task`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task).
Now, when the `Application` determined that the `echo` function should handle *Update A*, it creates a new task from `echo(Update A)`.
Immediately after that, it calls `Application.process_update(Update B)` and repeats the process for *Update B* without any further delay.
Both replies are sent **concurrently**.

Again, let's have a look at pseudocode:

```python
async def process_update(self, update):
    # self is the `Application` instance
    for group_number, handlers in self.handlers.items():
        # `handlers` is the list of handlers in group `group_number`
        for handler in handlers:
            if handler.check_update(update):
                # Here we *don't* `await`, such that the loop immediately continues
                asyncio.create_task(handler.handle_update(update))
                break  # at most one handler per group handles the update
```

This already helps for many use cases.
However, by using `block=False` in a handler, you can no longer rely on handlers in different groups being called one after the other.
Depending on your use case, this can be an issue.
Hence, PTB comes with a second option.

### `Application.concurrent_updates`

Instead of running single handlers in a non-blocking way, we can tell the `Application` to run the whole call of `Application.process_update` concurrently:

```python
Application.builder().token('TOKEN').concurrent_updates(True).build()
```

Now the `Application` will start `Application.process_update(Update A)` via `asyncio.create_task` and immediately afterwards do the same with *Update B*.
Again, pseudocode:

```python
while not application.update_queue.empty():
  update = await application.update_queue.get()
  asyncio.create_task(application.process_update(update))
```
This setting is *independent* of the `block` parameter of `Handler` and within `application.process_update` concurrency still works as explained above.

You can further customize concurrent handling `Application.process_update` also implement your own custom update processor by subclassing the [`BaseUpdateProcessor`](https://docs.python-telegram-bot.org/telegram.ext.baseupdateprocessor.html) interface class. Let's have a look at an example:
```python
class MyUpdateProcessor(BaseUpdateProcessor):
    async def do_process_update(self, update, coroutine) -> None:
        # This method is called for every update
        if update.callback_query:
            await asyncio.sleep(5)
        await coroutine

    async def initialize(self) -> None:
      pass

    async def shutdown(self) -> None:
      pass

Application.builder().token('TOKEN').concurrent_updates(MyUpdateProcessor(10)).build()
```
The above code processes every `callback_query` update with a delay of 5 seconds for up to 10 updates simultaneously. 
The psuedocode for this now looks something like this:
```python
while not application.update_queue.empty():
  update = await application.update_queue.get()
  coroutine = application.process_update(update)
  asyncio.create_task(my_update_processor.do_process_update(update, coroutine))
```
This is just an example of how to use the `BaseUpdateProcessor` class to handle updates in the way you want, there are endless possibilities to this.
For example, you can throttle update processing for specific users or ensure that inline queries are always processed sequentially.
See the [documentation](https://docs.python-telegram-bot.org/telegram.ext.baseupdateprocessor.html#telegram.ext.BaseUpdateProcessor) for more information.

**Note:** The number of concurrently processed updates is limited (the limit defaults to 4096 updates at a time).
This is a simple measure to avoid e.g. DDOS attacks

### `Application.create_task`

`Handler.block` and `Application.concurrent_updates` allow running handler callbacks or the entirety of handling an update concurrently.
In addition to that, PTB offers `Application.create_task` to run specific coroutine function concurrently.
`Application.create_task` is a very thin wrapper around [`asyncio.create_task`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task) that adds some book-keeping that comes in handy for using it in PTB.
Please consult the documentation of [`Application.create_task`](https://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.application.html#telegram.ext.Application.create_task) for more details.

This wrapper gives you fine-grained control about how you use concurrency in PTB.
The next section gives you an idea about why that is helpful.

## Tailor-made Concurrency

Even though `asyncio` is usually single-threaded, concurrent programming comes with a number of traps to fall into, and we'll try to give you a few hints on how to spot them.
However, this wiki article does not replace ~~your psychiatrist~~ a university lecture on concurrency.

Probably the biggest cause of issues of concurrency are shared states, and those issues are hard to fix.
So instead of showing you how to fix them, we'll show you how to avoid them altogether. More about that later.

**A fair warning:** In this section, we'll try to give you a simple talk (if that's possible) on a very complex topic.
Many have written about it before, and we're certainly less qualified than most.
As usual, we'll use an example to complement the text, and try to stay in the realm of what's important to you.

An example that is often used to illustrate this is that of a bank.
Let's say you have been hired by a bank to write a Telegram bot to manage bank accounts. The bot has the command `/transaction <amount> <recipient>`, and because many people will be using this command, you think it's a good idea to make this command run concurrently.
~~You~~ Some unpaid intern wrote the following (**BAD AND DANGEROUS**) callback function:

```python
async def transaction(update, context):
  bot = context.bot
  chat_id = update.effective_user.id
  source_id, target_id, amount = parse_update(update)

  await bot.send_message(chat_id, 'Preparing...')
  bank.log(BEGINNING_TRANSACTION, amount, source_id, target_id)

  source = bank.read_account(source_id)
  target = bank.read_account(target_id)

  source.balance -= amount
  target.balance += amount

  await bot.send_message(chat_id, 'Transferring money...')
  bank.log(CALCULATED_TRANSACTION, amount, source_id, target_id)

  bank.write_account(source)
  await bot.send_message(chat_id, 'Source account updated...')
  await bot.send_message(chat_id, 'Target account updated...')
  bank.write_account(target)

  await bot.send_message(chat_id, 'Done!')
  bank.log(FINISHED_TRANSACTION, amount, source_id, target_id)

application.add_handler(CommandHandler('transaction', transaction, block=False))
```

We skipped some of the implementation details, so here's a short explanation:

- `parse_update` extracts the user id's of the sender (`source_id`) and receiver (`target_id`) from the message
- `bank` is a globally accessible object that exposes the Python API of the banks operations
  - `bank.read_account` reads a bank account from the bank's database into a Python object
  - `bank.write_account` writes a bank account back to the bank's database
  - `bank.log` must be used to keep a log of all changes to make sure no money is lost

Sadly, ~~you~~ that damn intern fell right into the trap.
Let's say there are two morally corrupt customers, *Customer A* with *Account A* and *Customer B* with *Account B*, who both make a transaction simultaneously.
*Customer A* sends *Transaction AB* of *$10* to *Customer B*.
At the same time, *Customer B* sends a *Transaction BA* of *$100* to *Customer A*.

Now the `Application` starts two tasks, *Task AB* and *Task BA*, almost simultaneously.
Both tasks read the accounts from the database with the **same** balance and calculate a new balance for both of them.
In most cases, one of the two transactions will simply overwrite the other.
That's not too bad, but will at least be confusing to the customers.
However, each `await` gives control back to the event loop which may then continue another task.
Hence, the following may occur:

1. *Task AB* executes `bank.write_account(source)` and updates *Account A* with *-$10*
2. Before updating *Account B*, *Task AB* sends two messages and during that time, the event loop continues *Task BA*
3. *Task BA* executes `bank.write_account(source)` and updates *Account B* with *-$100*
4. Before updating *Account A*, *Task BA* sends two messages and during that time, the event loop continues *Task AB*
5. *Task AB* executes `bank.write_account(target)` and updates *Account B* with *+$10*
6. When *Task BA* is resumed again, it executes `bank.write_account(target)` and updates *Account A* with *+$100*

In the end, *Account A* is at *+$100* and *Account B* is at *+$10*.
Of course, this won't happen very often.
And that's what makes this bug so critical.
It will probably be missed by your tests and end up in production, potentially causing a lot of financial damage.

**Note:** This kind of bug is called a [race condition](https://en.wikipedia.org/wiki/Race_condition) and has been the source of many, many security vulnerabilities.
It's also one of the reasons why banking software is not written by unpaid interns.

To be fair, you probably don't write software for banks (if you do, you should already know about this), but this kind of bug can occur in much simpler situations.
While in this case, the shared state is the `bank` object, it can take many forms.
A database, a `dict`, a `list` or any other kind of object that is modified by more than one task.
Depending on the situation, race conditions are more or less likely to occur, and the damage they do is bigger or smaller, but as a rule of thumb, they're bad.

As promised in the first paragraph, let's discuss how to avoid such situations.
That's not always as easy as it is in this case, but we're lucky:

1. Our set of tools is very limited - `Application.create_task` is the only `asyncio` tool we're using
2. Our goals are not very ambitious - we only want to speed up our I/O

There are two relatively simple steps you have to follow.
First, identify those parts of the code that **must** run sequentially (the opposite of *in parallel* or *concurrently*).
Usually, that is code that fits **at least one** of these criteria:

1. *Modifies* shared state
2. *Reads* shared state and *relies on* it being correct
3. *Modifies* local state (e.g. a variable used later in the same function)

Make sure you have a good idea what *shared state* means
Don't hesitate to do a quick Google search on it.

Let's go through our bank example line by line and note which of the criteria it matches:

```python
async def transaction(update, context):
  bot = context.bot
  chat_id = update.effective_user.id  # 3
  source_id, target_id, amount = parse_update(update)  # 3

  await bot.send_message(chat_id, 'Preparing...')  # None
  bank.log(BEGINNING_TRANSACTION, amount, source_id, target_id)  # None

  source = bank.read_account(source_id)  # 2, 3
  target = bank.read_account(target_id)  # 2, 3

  source.balance -= amount  # 3
  target.balance += amount  # 3

  await bot.send_message(chat_id, 'Transferring money...')  # None
  bank.log(CALCULATED_TRANSACTION, amount, source_id, target_id)  # None

  bank.write_account(source)  # 1
  await bot.send_message(chat_id, 'Source account updated...')  # None
  await bot.send_message(chat_id, 'Target account updated...')  # None
  bank.write_account(target)  # 1

  await bot.send_message(chat_id, 'Done!')  # None
  bank.log(FINISHED_TRANSACTION, amount, source_id, target_id)  # None

application.add_handler(CommandHandler('transaction', transaction, block=False))
```

**Note:**
One could argue that `bank.log` modifies shared state.
However, logging libraries are usually thread-safe and it's unlikely that the log has a critical functional role.
It's not being read from in this function, and let's assume it's not being read from anywhere else in the code, so maybe consider this an exception to the rule.
Also, for the sake of this example, it'd be boring if only `bot.sendMessage` would be safe to run in parallel.
However, we will keep this in mind for the next step.

As you can see, there's a pretty obvious pattern here:
`bot.send_message` and `bank.log` are not matching any criteria we have set for strictly sequential code.
That means we can run this code asynchronously without risk.
Therefore, the second step is to extract that code to separate functions and run only them concurrently.
Since our async code parts are all very similar, they can be replaced by a single function.
We could have done that before, but then this moment would've been less cool.

```python
async def log_and_notify(action, amount, source_id, target_id, chat_id, message):
  bank.log(action, amount, source_id, target_id)
  await bot.send_message(chat_id, message)

async def transaction(update, context):
  chat_id = update.message.chat_id  # 3
  source_id, target_id, amount = parse_update(update)  # 3

  context.application.create_task(
    log_and_notify(
      BEGINNING_TRANSACTION,
      amount,
      source_id,
      target_id,
      chat_id,
      'Preparing...',
    ),
    update=update
  )

  source = bank.read_account(source_id)  # 2, 3
  target = bank.read_account(target_id)  # 2, 3

  source.balance -= amount  # 3
  target.balance += amount  # 3

  context.application.create_task(
    log_and_notify(
      CALCULATED_TRANSACTION,
      amount,
      source_id,
      target_id,
      chat_id,
      'Transferring money...'
    ),
    update=update
  )

  bank.write_account(source)  # 1
  bank.write_account(target)  # 1


  context.application.create_task(
    log_and_notify(
      FINISHED_TRANSACTION,
      amount,
      source_id,
      target_id,
      chat_id,
      'Done!',
    ),
    update=update
  )

application.add_handler(CommandHandler('transaction', transaction, block=True))
```

**Note:** You might have noticed that we moved `bank.log` before `bot.send_message`, so the log entries will be in order *most of the time*, assuming the database operations take long enough for the log to complete.

**Note:** It's likely that `bank.read_account` and `bank.write_account` require some I/O operations to interact with the banks database.
You see that it's not always possible to write code concurrently, at least with this simplified method. Read about [Transactions](https://en.wikipedia.org/wiki/Database_transaction) to learn how databases solve this in "real life".

By separating the strictly sequential code from the concurrent code, we made sure that no race conditions can occur.
The `transaction` function won't be executed concurrently anymore, but we still managed to gain some substantial performance boost over completely sequential code, because the logging and user notification is now run in parallel.

That's basically it for now.
For the very end, here are a few helpful guidelines for writing concurrent code:

- Avoid using shared state whenever possible
- Write self-contained ([pure](https://en.wikipedia.org/wiki/Pure_function)) functions
- When in doubt, make it sequential
