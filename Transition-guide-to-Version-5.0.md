## JobQueue
We did some serious work on the `telegram.ext.JobQueue` class. The changes are similar to the changes made to the `telegram.ext.Dispatcher` class in version 4. The [[Extensions - JobQueue|Extensions---JobQueue]] article has been updated with the changes.

## Botan
Botan was moved from `telegram.utils.botan` to `telegram.contrib.botan`

## New: ConversationHandler
The `telegram.ext.ConversationHandler` class has been added. It implements a [state machine](https://en.wikipedia.org/wiki/Finite-state_machine) and replaces the old `state_machine_bot.py` example with the new [`conversationbot.py`](https://github.com/python-telegram-bot/python-telegram-bot/blob/master/examples/conversationbot.py) example. [Read the documentation](http://python-telegram-bot.readthedocs.io/en/latest/telegram.ext.conversationhandler.html) for more information.