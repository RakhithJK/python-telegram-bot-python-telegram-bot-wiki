This page describes advanced use cases for the filters used with `MessageHandler` from `telegram.ext`.

# Combining filters
When using `MessageHandler` it is sometimes useful to have more than one filter. This can be done using so called bit-wise operators. In Python those operators are `&`, `|` and `~` meaning AND, OR and NOT respectively. Since version 13.1 filters support `^` for XOR.
## Examples
#### Message is either video, photo, or document (generic file)
``` python
from telegram.ext import MessageHandler, Filters

handler = MessageHandler(Filters.video | Filters.photo | Filters.document, 
                         callback)
```

#### Message is a forwarded photo
``` python
handler = MessageHandler(Filters.forwarded & Filters.photo, callback)
```

#### Message is text and contains a link
``` python
from telegram import MessageEntity

handler = MessageHandler(
    Filters.text & (Filters.entity(MessageEntity.URL) |
                    Filters.entity(MessageEntity.TEXT_LINK)),
    callback)
```

#### Message is a photo and it's not forwarded
``` python
handler = MessageHandler(Filters.photo & (~ Filters.forwarded), callback)
```

# Custom filters
It is also possible to write our own filters. In essence, a filter is simply a function that receives either a `Message` instance or a `Update` instance and returns either `True` or `False`. This function has to be implemented in a new class that inherits from either `MessageFilter` or `UpdateFilter`, which allows it to be combined with other filters. If the combination of all filters evaluates to `True`, the message will be handled.

The difference between `UpdateFilter` and `MessageFilter` is that the `filter` function of the former will receive the `update`, allowing e.g. to differentiate between channel post updates and message updates, while the `filter` function of the latter will receive the `update.effective_message`.

Say we wanted to allow only those messages that contain the text "python-telegram-bot is awesome", we could write a custom filter as so:

```python
from telegram.ext import MessageFilter

class FilterAwesome(MessageFilter):
    def filter(self, message):
        return 'python-telegram-bot is awesome' in message.text

# Remember to initialize the class.
filter_awesome = FilterAwesome()
```

The class can of course be named however you want, the only important things are:
- The class has to inherit from `MessageFilter` or `UpdateFilter`
- It has to implement a `filter` method
- You have to create an instance of the class

The filter can then be used as:
```python
awesome_handler = MessageHandler(filter_awesome, callback)
```

## `Filters` and `CallbackContext`

You may have noticed that when using `Filters.regex`, the attributes `context.matches` and `context.match` are set to the corresponding matches. To achieve something like this for your custom filter, you can do the following:

1. Set `self.data_filter=True` for your filter.
2. If the update should be handled return a dictionary of the form `{attributen_name: value}`. This dict will be merged with the internal dict of the `context` argument making `value` available as `context.attribute_name`. This currently works with `MessageHandler`, `CommandHandler` and `PrefixHandler`, which are the only handlers that accept filters.

If you want this to work with your custom handler, make sure that `YourHandler.collect_additional_context` does something like

```python
if isinstance(check_result, dict):
    context.update(check_result)
```
