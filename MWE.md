# How to write a Minimal Working Example

If you read this, you probably were asked to provide a minimal working example (MWE) in the [user group](https://t.me/pythontelegrambotgroup) or the [issue tracker](https://github.com/python-telegram-bot/python-telegram-bot/issues) of the [python-telegram-bot](https://python-telegram-bot.org) library.

So here is what that means:

## Example

When trying to help you with a problem, it's often helpful to see your code instead of a vague description of the issue. Of course, a better description often also helps (see [[this article|Ask-Right]] on asking good technical questions).

## Working
In order for the example to actually be helpful, it must work. This means that it:

### Is ready to run
Code that's not runnable is not very useful to see an issue - at least most times. Make sure that whoever is trying to help you only needs to plug in a bot token, run the script and send `/start` to the bot for the example to run.

In particular this means that

* all necessary imports are included
* no undefined functions are called or undefined modules imported
* apart from PTB no additional dependencies need to be installed

### Reproduces the issue
If the provided example doesn't show the described problem, it's not much of an example.

## Minimal
Make sure that your example contains everything needed for the problem to show - and nothing more! Nothing delays finding a bug more than hundreds of unrelated lines of code. Your example should fit in a single short file. Reducing your problematic code to such a minimal example often times already helps to identify the actual issue.

## Publishing
Be sure to remove your bot token before publishing the MWE!
Always send it along with a concise description of what exactly the problem is and how it can be reproduced with your MWE.

As a final note, if you were asked for an MWE in the user group, please post your code using a pastebin, e.g. https://hastebin.com or https://pastebin.com, instead of posting it as plain text in the user group.