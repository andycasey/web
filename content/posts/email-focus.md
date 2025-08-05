---
title: Keep your email out of focus
description: "You'll never guess what the best email productivity hack is!"
date: 2025-08-03
authors:
  - name: Andy Casey
    link: https://astrowizici.st
    image: ../../me.png
tags:
  - productivity
  - automation
  - email
---

The easiest way to be more productive with email is to not use it. 

But sometimes you have to. So maybe a compromise would be just not to use it until you have to. 

I don't have notifications appear on my laptop, but sometimes I do notice the little red badge on my email client, and it distracts me to check that email. And sometimes if I have my email client closed, I find I can get deeper into research without distractions.

For these reasons, I've come up with the best productivity hack ever.

{{% steps %}}

### Create a place for the script and logs

```bash
mkdir -p ~/scripts ~/logs
```

### Create the script

Now let's create a script at `~/scripts/kill_spark_if_inactive.sh`

The following script will check to see if your email client is active. If it is not active, it will kill your email client. If it is active, it will do nothing.

I use [Spark](https://sparkmailapp.com/) as my email client, but you can change the script to suit your own needs.

```bash
#!/bin/bash

# Get the frontmost (active) application
ACTIVE_APP=$(osascript -e 'tell application "System Events" to get name of first application process whose frontmost is true')

# Check if Spark is running
SPARK_RUNNING=$(pgrep -f "Spark")

# Only kill Spark if it's running AND not the active application
if [ ! -z "$SPARK_RUNNING" ] && [ "$ACTIVE_APP" != "Spark" ]; then
    echo "$(date): Killing Spark (not active). Active app was: $ACTIVE_APP" >> ~/logs/spark_kill.log
    pkill -f "Spark"
else
    if [ -z "$SPARK_RUNNING" ]; then
        echo "$(date): Spark not running, nothing to kill" >> ~/logs/spark_kill.log
    else
        echo "$(date): Spark is active, not killing. Active app: $ACTIVE_APP" >> ~/logs/spark_kill.log
    fi
fi
```

### Make the script executable

```bash
chmod +x ~/scripts/kill_spark_if_inactive.sh
```

### Set up our cron jobs

Now let's set up a cron job.

```bash
crontab -e
```

And add the following lines to your crontab file:

```bash
0   8 * * * /usr/bin/open -a Spark
30  8 * * * ~/scripts/kill_spark_if_inactive.sh
30 16 * * * /usr/bin/open -a Spark
0  17 * * * ~/scripts/kill_spark_if_inactive.sh
```

{{% /steps %}}

This will open Spark (my email client) at 8:00 AM, and then kill it at 8:30 AM if it is not the active application. It will then open Spark again at 4:30 PM, and kill it again at 5:00 PM if it is not the active application.

You're welcome in advance.