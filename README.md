# ed-server-monitoring-php-and-bash
Server monitoring with PHP and Bash Scripts

by [Server monitoring with PHP and Bash Scripts](https://medium.com/winkhosting/server-monitoring-with-php-and-bash-scripts-c078616a6e2f)

## Why do I require a custom solution to monitor my servers?

I like your question, your question is one of those… good questions. In my case, because I want something specialized for our servers and we want to release it as a product to our customers.

Your needs may be different but I recommend you explore this option, despite the fact that there are lots of solutions already available and much more advanced (bearing in mind that the greater the number of features, the complexity increases and the technical and knowledge requirements as well).


## The basic components

In this post we will cover the process of creating the Bash script that will capture the information from our server and send it to a central repository. We will also explore a web service that will be receiving the telemetry data and it will also store it as plain files on a directory.

We will not cover the presentation of information. I leave it to your imagination this time. (You have the power to do it boy!).

Let’s start with the bash script:

## Requirements

Our servers use Linux, we will create this script using Ubuntu 22.04 LTS. and the script will monitor the following information:

- Load and Uptime every 10 seconds
- Storage every 24 hours
- RAM usage every 30 seconds
- Top 10 processes using CPU every 5 minutes

The commands to be used to capture this information will be the following:
Load and Uptime

We will check the load and uptime in a simple way with the “uptime” command from the server command console:

```
> uptime

 09:32:48 up 296 days,  3:01,  2 users,  load average: 1.92, 1.23, 0.97
```

This command displays the time the server has been running since the last reboot, the number of active users, and the average processing load for the last 1, 5, and 15 minutes.

## Storage

Storage can be queried as follows with the “df -h” command:

```
> df -h

Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        384M     0  384M   0% /dev
tmpfs           403M     0  403M   0% /dev/shm
tmpfs           403M   42M  362M  11% /run
tmpfs           403M     0  403M   0% /sys/fs/cgroup
/dev/sda         25G   12G   12G  49% /
/dev/sdb         25G  4.2M   24G   1% /home2
/dev/loop0      915M  804K  867M   1% /tmp
tmpfs            81M     0   81M   0% /run/user/0
```

When executed, the detected storage devices are shown with their respective mount point, total and used capacity in Megabytes.

## RAM usage

RAM usage can be checked as follows with the “free -m” command:

```
> free -m

              total        used        free      shared  buff/cache   available
Mem:            804         533          82          50         188         102
Swap:             0           0           0
```

The command will display the server’s RAM, Swap usage, free memory and cache

## Top 10 processes that use CPU

This command is more elaborate, it requires the combination of three basic commands to sort, filter and display the 10 processes with the highest CPU usage like this:

```
> ps -eo pcpu,pid,user,args | sort -k 1 -r | head -10

%CPU     PID USER     COMMAND
 0.4      75 root     [kswapd0]
 0.3 3623591 mysql    /usr/sbin/mysqld
 0.1 3831215 root     lfd - sleeping
 0.1    1079 root     /usr/libexec/platform-python -Es /usr/sbin/tuned -l -P
 0.0       9 root     [mm_percpu_wq]
 0.0     839 root     /usr/lib/systemd/systemd-logind
 0.0     838 root     /usr/sbin/NetworkManager --no-daemon
 0.0     831 root     [ext4-rsv-conver]
 0.0     830 root     [jbd2/loop0-8]
```

First we will use the “ps” command to display cpu usage, process id, user, and arguments used. Next we order the list by the first column with the “sort” command and finally we filter to leave only the first 10 records with the “head” command.

## The file

Let’s start by creating a file named “servermon.sh”. You can use the name you want, it is only necessary to keep it in mind during the creation process. The initial file will look like this:

servermon.sh (init)

```
#!/bin/bash
```

The first line indicates the interpreter to use, normally this value should not be changed. Now we go with the initial capture of information.

## Data collection

To capture the result of the execution of the commands that we are going to use, we will send the console output to a text file like this:

servermon.sh (data collection)

```
#!/bin/bash

# Temp storage dir
DIRTMP=/tmp

# uptime and load
uptime > $DIRTMP/uptime.log

# storage
  df -h > $DIRTMP/storage.log

# memory usage
free -m > $DIRTMP/memoryusage.log

# top 10 cpu
ps -eo pcpu,pid,user,args | sort -k 1 -r | head -10 > $DIRTMP/top10cpu.log
```

We also create a variable that contains the temporary directory to save the captured files, you can use the directory of your choosing, in this case we will use /tmp and will be updated for each data submission, it is not necessary to save them indefinitely.

## Capture loop

The capture loop will allow us to check the information periodically, for each type of query we will define a customizable interval since it is not necessary (or at least in this case) to check the storage every 3 seconds. But we are interested in checking the server load at shorter intervals.

Let’s define the timers for collecting information like this:

servermon.sh (loop y timers)

```
#!/bin/bash

# Temp storage dir
DIRTMP=/tmp

# Timers (in seconds)
# Storage timer
STORAGETIMER=3600
# RAM timer
RAMTIMER=60
# CPU timer
CPUTOPTIMER=300
# Uptime timer
UPTIMETIMER=5

# Tickers
STORAGETICKER=0
RAMTICKER=0
CPUTOPTICKER=0
UPTIMETICKER=0

# Our main loop
while true
do

  # Ticker update every second
  ((STORAGETICKER += 1))
  ((RAMTICKER += 1))
  ((CPUTOPTICKER += 1))
  ((UPTIMETICKER += 1))

  # If ticker reaches the limit to report information
  if [ $STORAGETICKER == $STORAGETIMER ]; then
    # ticker reset
    ((STORAGETICKER = 0))
    # Capture info
    df -h > $DIRTMP/storage.log
    rm -f $DIRTMP/storage.log
  fi

  # Check RAM.
  if [ $RAMTICKER == $RAMTIMER ]; then
      ((RAMTICKER = 0))
      free -m > $DIRTMP/memoryusage.log
      rm -f $DIRTMP/memoryusage.log
  fi

  # Check Top 10 + CPU
  if [ $CPUTOPTICKER == $CPUTOPTIMER ]; then
      ((CPUTOPTICKER = 0))
      ps -eo pcpu,pid,user,args | sort -k 1 -r | head -10 > $DIRTMP/top10cpu.log
      rm -f $DIRTMP/top10cpu.log
  fi

  # Uptime and load
  if [ $UPTIMETICKER == $UPTIMETIMER ]; then
      ((UPTIMETICKER = 0))
      uptime > $DIRTMP/uptime.log
      rm -f $DIRTMP/uptime.log

  fi

  # Wait a second...
  sleep 1

done
```

As you can see, running this script will periodically capture CPU, RAM, Uptime/Load and Storage information to a flat file.

At the end of each cycle, the file that contains the information is deleted. If you only want to keep the files to check them later, you can remove or comment the deletion line and assign a variable text string to the file name (the date and time, for example) to maintain files with historical data. Since in this case we plan to send the information to a central server, we will not store files locally.

Now we are going to create the web service with PHP to receive the information and we will adjust the bash script to send the files.

## PHP endpoint/script

We are going to create a “hack and slash” kind of script as a demo, simple, fast to receive the files and store them.

## Requirements

You must have a development environment compatible with PHP 8.x to create this script, keep in mind that our bash script must also have access within the local network or Internet to the URL of the PHP script we are going to create to send the files to it.

## The file

In our case we create a /storeinfo/ directory at the root of our web server directory and we will create a file and call it index.php, also create a /files directory inside /storeinfo/, this will be used to save the received files.

The index.php file will look like this:

```
<?php

//Security token to authenticate the access to this script.
$_TOKEN = "f66dc5a4-f668-40d6-9b31-7d5d7c947985";

//Received files storage path.
$_SAVEPATH = __DIR__ . "/files";

//Checks if a token was received over POST, if not the script ends.
if (!isset($_POST["token"])) {
    echo "Token not found, you need a valid token.";
    exit;
}

//Checks if the token is valid, if not the script ends.
if ($_POST["token"] != $_TOKEN) {
    echo "The token is not valid.";
    exit;
}

//Checks if there is a file in the request to be stored.
if (!isset($_FILES)) {
    echo "There is no file to be stored, please check that the request was made over POST and that it includes a file.";
    exit;
}

//Creates the full path where the file will be stored, it adds date and time to the filename.
$_SAVEPATH = $_SAVEPATH . "/" . date("Ymdhis") . "_" . basename($_FILES['data']['name']);


//Moves the file to the $_SAVEPATH, and shows a message according to the operation result.
if (!move_uploaded_file($_FILES['data']['tmp_name'], $_SAVEPATH)) {
    echo "It was not possible to move the file to the path {$_SAVEPATH}";
    exit;
} else {
    echo "File saved at {$_SAVEPATH}";
    exit;
}
```

First we define a token to validate that the client sending the information is authorized and a save directory to store the files received.

The received files will be saved with a name format like this “<yyymmddhhis>_<original name>” in the /files directory.

Now that we have our php file to receive data, let’s adjust the bash script so that it can send data to it. For this we are going to add the curl command as shown below:

## cURL (POST) Command

```
curl -F "data=@$DIRTMP/storage.log" -F "token=$TOKEN" http://127.0.0.1/storeinfo/index.php
```

The command objective is to send a POST request adding the content of the file using data=<file path> and token=<token for authentication> to the URL http://127.0.0.1/storeinfo/index.php

The -F parameter tells cURL to treat each of the parameters as part of a form to be submitted using POST.

The final bash script will look like this:


### servermon.sh (final)

```
#!/bin/bash

# Temp storage dir
DIRTMP=/tmp

# Timers (in seconds)
# Storage timer
STORAGETIMER=3600
# RAM timer
RAMTIMER=60
# CPU timer
CPUTOPTIMER=300
# Uptime timer
UPTIMETIMER=5

# Tickers
STORAGETICKER=0
RAMTICKER=0
CPUTOPTICKER=0
UPTIMETICKER=0

# Our main loop
while true
do

  # Ticker update every second
  ((STORAGETICKER += 1))
  ((RAMTICKER += 1))
  ((CPUTOPTICKER += 1))
  ((UPTIMETICKER += 1))

  # If ticker reaches the limit to report information
  if [ $STORAGETICKER == $STORAGETIMER ]; then
    # ticker reset
    ((STORAGETICKER = 0))
    # Capture info
    df -h > $DIRTMP/storage.log
    curl -F "data=@$DIRTMP/storage.log" -F "token=$TOKEN" $URL
    rm -f $DIRTMP/storage.log
  fi

  # Check RAM.
  if [ $RAMTICKER == $RAMTIMER ]; then
      ((RAMTICKER = 0))
      free -m > $DIRTMP/memoryusage.log
      curl -F "data=@$DIRTMP/memoryusage.log" -F "token=$TOKEN" $URL
      rm -f $DIRTMP/memoryusage.log
  fi

  # Check Top 10 + CPU
  if [ $CPUTOPTICKER == $CPUTOPTIMER ]; then
      ((CPUTOPTICKER = 0))
      ps -eo pcpu,pid,user,args | sort -k 1 -r | head -10 > $DIRTMP/top10cpu.log
      curl -F "data=@$DIRTMP/top10cpu.log" -F "token=$TOKEN" $URL
      rm -f $DIRTMP/top10cpu.log
  fi

  # Uptime and load
  if [ $UPTIMETICKER == $UPTIMETIMER ]; then
      ((UPTIMETICKER = 0))
      uptime > $DIRTMP/uptime.log
      curl -F "data=@$DIRTMP/uptime.log" -F "token=$TOKEN" $URL
      rm -f $DIRTMP/uptime.log

  fi

  # Wait a second...
  sleep 1

done
```

For each type of information to be reported, a new line was added so that the file can be sent via cURL to the URL of the service where our PHP file is located.

Now we are ready, with this it would be enough to capture the information and send it to a central server for storage or processing.

You can try as follows:

## Bash script

```
# open a terminal on your linux test server (where the script servermon.sh is located and run:
chmod +x servermon.sh
# now start the script:
./servermon.sh
```

If all goes well (and I hope it goes well…), the result of the submissions will be displayed after a few seconds (depending on the submission interval that you set on the bash script).

Now, on the web server, you can go into the /files directory and you will find files with names like these:

```
/files
  20230424043142_uptime.log
  20230424043242_uptime.log
```

Each file will contain the information reported by the server for each type of monitored resource.

At this point what is left is to process the information within the files and decide what to do with it on the central server.
