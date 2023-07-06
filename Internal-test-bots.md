**Note: This guide is of no use to the regular user.**

### Variables:
```
CI = AppVeyor|Travis
py_platform = CPython|PyPy
py_version = 2.7|3.4|3.5|3.6|3.7
bot_username = ptb_{CI.lower()}_{py_platform.lower()}_{py_version.replace('.', '')}_bot
```
## Script
[Here](https://gist.github.com/jsmnbom/2e8044ca5cc55813a0e0380ad375b320) is a script that does all the setup below in a semi automated way using Telethon. Also check [this version](https://gist.github.com/Bibo-Joshi/75f135edf1ca3530decf4c2ae06bd699), which was updated for the animated sticker sets and automatically creates a new super group for the bot to be added to (see [#1919](https://github.com/python-telegram-bot/python-telegram-bot/pull/1919)). ⚠️ The script needs to be updated to create a forum test group for each bot and make the bot admin (with right to edit forums!) in that group. Teltehons `ToggleForumRequest` is helpful for converting groups to forums.

**Note:** It doesn't create the video sticker set yet, but the test suite creates that on the fly, if necessary.


## Setup a new bot

`#` means *send a message to @BotFather*  
`%` means *click button in inlinekeyboard*  

### Create the bot

`# /newbot`  
`# Python-telegram-bot tests on {CI} using {py_platform} {py_version}` *(Bot name)*  
`# {bot_username}` *(Bot username)*  

The token for the new bot will be shown.

### Allow the bot to talk to you
Search for the bot, and press the `Start` button

### Set texts in case a user stumbles upon the bot

`# /set_description`  
`# @{bot_username}`  
`# This bot is only for running tests for python-telegram-bot and has no actual functionality.`  

`# /set_abouttext`  
`# @{bot_username}`  
`# This bot is only for running tests for python-telegram-bot and has no actual functionality.`  

### Make sure it can join groups

`# /setjoingroups`  
`# @{bot_username}`  
`# Enable`  

### Add it to the developer group
`>>> telegram.Bot() - Developers`

### Add it to the testing channel
[`>>> telegram.Bot() - Tests`](https://t.me/pythontelegrambottests)

### Make sure it can not join groups anymore

`# /setjoingroups`  
`# @{bot_username}`  
`# Disable` 

### Turn on inline so we can create a game for the bot

`# /set_inline`  
`# @{bot_username}`  
`# This bot is only for running tests.`  

### Create a game for the bot

`# /newgame`  
`# OK`  
`# Accept`  
`# @{bot_username}`  
`# Python-telegram-bot test game`  
`# A no-op test game, for python-telegram-bot bot framework testing.`  
`Upload tests/data/game.png`  
`Upload tests/data/game.gif`  
`# test_game`  

### Payment
Now we need to setup payment. This can only be done via the beta /mybots interface.

`# /mybots`  
`% @{bot_username}`  
`% Payments`  
`% Stripe` *(might have a flag in front, depending on your country)*  
`% Connect Stripe Test`  
`Press start in @StripeTestBot`  
`% Authorize`  
`Press "Skip this account form" at the top of your browser`  
`Go back to @StripeTestBot and make sure everything went well`  
`Go back to @BotFather`  

The payment provider token should be displayed in the last message.

### Create a sticker set for the bot

We need to use the bot api to do this.
```
me_id = YOURUSERID
sticker_set_name = 'test_by_{username}
sticker_set_title = 'Test',
sticker = 'tests/data/telegram_sticker.png',
sticker_emoji = '😄'

bot = telegram.Bot(token)
with open(sticker, 'rb') as f:
    assert bot.create_new_sticker_set(me.id, sticker_set_name, sticker_set_title,
                                      f, sticker_emoji)
```

### Create an animated sticker set for the bot

We need to use the bot api to do this.
```
me_id = YOURUSERID
animated_sticker_set_name = 'test_by_{username}
animated_sticker_set_title = 'Test',
animated_sticker = 'tests/data/telegram_animated_sticker.png',
sticker_emoji = '😄'

bot = telegram.Bot(token)
with open(animated_sticker, 'rb') as f:
    assert bot.create_new_sticker_set(me.id, animated_sticker_set_name, animated_sticker_set_title,
                                      f, sticker_emoji)
```

### Create a video sticker set for the bot

Similar to above
