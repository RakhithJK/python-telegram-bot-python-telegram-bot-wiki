### Introduction
You will need a VPS (or dedicated server) first. Check out the list at [[Where to host Telegram Bots|Where-to-host-Telegram-Bots#vps]] if you don't have one already.

### First login
Your login details should contain
- The IP address of your server
- Your username (usually `root`)
- Your password

You should also receive some information on the type of server you got, including Operating System, RAM, storage and monthly traffic.

Once you received that information, you can connect to the server via SSH. 

> Note that although the explanations below mention entering your password, it is strongly recommended to authenticate using a [SSH public key](https://www.ssh.com/academy/ssh/public-key-authentication) instead of the password. You can read [this explanation](https://www.ssh.com/academy/ssh/keygen) on how to generate keys and send the public key to the server, or check your VPS provider's documentation.

#### Linux
Run the following command in the terminal and replace `<user>` with your username and `<ip>` with your servers IP address:

```
ssh <user>@<ip>
```

Confirm that you want to trust the host and enter the password if you are asked to do so.

#### Windows
Install [puTTY](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html) and start it. 

In the field *Host Name (or IP address)* enter the IP address of your server. As the connection type, select *SSH* and set *Port* to *22*. You can save these settings my entering a name in the field below *Saved Sessions* and clicking *Save*. Then, click *Open* and enter your username and password when asked to do so.

### Setup

First, install the Python package manager `pip`.
```
apt-get update && apt-get install python3-pip
```

Now, install the `python-telegram-bot` library:
```
pip3 install python-telegram-bot
```

Finally, confirm the installation:
```
python3 -c "import telegram;print(telegram.__version__)"
```

Of course, this is the easy way; the correct way (especially if you have multiple Python projects on your server) is to install a **virtual environment** (like `pipenv` or `poetry`) first.

### Upload your bot's files
Now you can upload your bot to the server. There are multiple ways to do that, one way is to use [FileZilla](https://filezilla-project.org/download.php?type=client). Install it (if you're on Linux, chances are you already have it) and start it. Open the *Server Manager* and create a new server with the button on the left. Give it a nice name, then go to the right and fill in the fields:

- *Server:* Your servers IP address
- *Port:* 22
- *Protocol:* SFTP
- *Connection type:* Normal
- *Username:* Your username
- *Password:* Your password

Now, click *Connect*. You will probably see the `/root` directory (your user's home directory) on the right and your local files on the left. Create a directory for your bot and upload all the files needed by your bot into that directory.

### Start your bot
To run your bot, connect to your server again via SSH (or go back to the connection) and `cd` into the directory you created. You could now immediately start the bot, but then it would stop working once you disconnect from the server. There are again several ways to make sure that doesn't happen:

1. Use a terminal multiplexer like `screen` or `tmux`.
2. Run your bot as a systemd service.

#### Running your bot in a terminal multiplexer

`screen` is called a "terminal multiplexer". It creates *virtual terminals* that you can attach to and detach from and that can run processes without you being logged in.

Create a new *screen* and attach to it:
```
screen -S mybot
```

Start the bot:
```
python3 bot.py
```

Detach from the *screen* by holding <kbd>CTRL</kbd> and pressing <kbd>A</kbd>, then <kbd>D</kbd>. You can now disconnect from the server by typing `exit` if you want. 

To re-attach to the *screen* after you logged back in:
```
screen -r mybot
```
or
```
screen -d -r mybot
```

A known alternative to `screen` is [`tmux`](https://github.com/tmux/tmux/wiki).

#### Running your bot as a systemd service

Choosing this option will mean that your Python program will be running in the background. Another upside to this option is that you can set it up to start at system boot, which means that if you reboot your server (e.g. to apply system upgrades), your bot will start automatically afterwards.

1. Install [`python-dotenv`](https://pypi.org/project/python-dotenv/).
2. Create the `.env` file and put your bot-related environment variables in there (you didn't put your bot token right into your code, hopefully!). If you're using Git, make sure `.env` is in `.gitignore` so it doesn't get checked in.
3. Follow e.g. [this tutorial](https://blog.abhinandb.com/run-any-executable-as-systemd-service/) to run your Python script as a service. If you're using a virtual environment (see [this external article](https://docs.python-guide.org/dev/virtualenvs/) for details), you have to locate the Python executable first. To do that, activate the virtual environment on the remote server and enter `which python` in your bash console. You will get a path to the Python instance you will need to use when configuring systemd.

Steps 1 and 2 are optional if you're not using any virtual environment, but it's better to go through them anyway because if you set the environment variables manually, they will be lost after system reboot.

## What to read next?
If you plan on hosting multiple bots on your server, it's recommended to use `virtualenv`. It allows you to install and upgrade Python modules via `pip` for one project, without worrying how it affects other projects on the server. Read [this external article](http://docs.python-guide.org/dev/virtualenvs/) for more information.

Learn about how to use a webhook for your bot in [[this article|Webhooks]].

You might also read the article on [[Performance Optimizations|Performance-Optimizations]] if you didn't read it yet.