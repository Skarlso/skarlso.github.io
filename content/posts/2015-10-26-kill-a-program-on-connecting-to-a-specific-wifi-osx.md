+++
author = "hannibal"
categories = ["devops", "problem solving"]
date = "2015-10-26"
tags = ["launchd", "osx"]
title = "Kill a Program on Connecting to a specific WiFi – OSX"
url = "/2015/10/26/kill-a-program-on-connecting-to-a-specific-wifi-osx/"

+++

Hi folks.

If you have the tendency, like me, to forget that you are on the corporate VPN, or leave a certain software open when you bring your laptop to work, this might be helpful to you too.

It's a small script which kills a program when you change your Wifi network.

Script:

~~~bash

#!/bin/bash

function log {
    directory="/Users/<username>/wifi_detect"
    log_dir_exists=true
    if [ ! -d $directory ]; then
        echo "Attempting to create => $directory"
        mkdir -p $directory
        if [ ! -d $directory ]; then
            echo "Could not create directory. Continue to log to echo."
            log_dir_exists=false
        fi
    fi
    if $log_dir_exists ; then
        echo "$(date):$1" >> "$directory/log.txt"
    else
        echo "$(date):$1"
    fi
}

function check_program {
    to_kill="[${1::1}]${1:1}"
    log "Checking if $to_kill really quit."
    ps=$(ps aux |grep "$to_kill")
    log "ps => $ps"
    if [ -z "$ps" ]; then
	# 0 - True
        return
    else
	# 1 - False
        return 1
    fi
}

function kill_program {
    log "Killing program"
    `pkill -f "$1"`
    sleep 1
    if ! check_program $1 ; then
	log "$1 Did not quit!"
    else
	log "$1 quit successfully"
    fi
}

wifi_name=$(networksetup -getairportnetwork en0 |awk -F": " '{print $2}')
log "Wifi name: $wifi_name"
if [ "$wifi_name" = "<wifi_name>" ]; then
    log "On corporate network... Killing Program"
    kill_program "<programname>"
elif [ "$wifi_name" = "<home_wifi_name>" ]; then
    # Kill <program> if enabled and if on <home_wifi> and if Tunnelblick is running.
    log "Not on corporate network... Killing <program> if Tunnelblick is active."
    if ! check_program "Tunnelblick" ; then
	log "Tunnelblick is active. Killing <program>"
	kill_program "<program>"
    else
	log "All good... Happy coding."
    fi
else
    log "No known Network..."
fi
~~~

Now, the trick is, on OSX to only trigger this when your network changes. For this, you can have a 'launchd' daemon, which is configured to watch three files which relate to a network being changed.

The script sits under your ~/Library/LaunchAgents folder. Create something like, com.username.checknetwork.plist.

~~~xml

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" \
 "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>ifup.ddns</string>

  <key>LowPriorityIO</key>
  <true/>

  <key>ProgramArguments</key>
  <array>
    <string>/Users/username/scripts/ddns-update.sh</string>
  </array>

  <key>WatchPaths</key>
  <array>
    <string>/etc/resolv.conf</string>
    <string>/Library/Preferences/SystemConfiguration/NetworkInterfaces.plist</string>
    <string>/Library/Preferences/SystemConfiguration/com.apple.airport.preferences.plist</string>
  </array>

  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
~~~

Now, when you change your network, to whatever your corporate network is, you'll kill Sublime.

Hope this helps somebody.

Cheers,

Gergely.