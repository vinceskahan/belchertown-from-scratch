>[!Note]
>This is obviously a draft


# Belchertown websockets from scratch
The intent of this document is to do a step-by-step walkthrough of how to get the weewx Belchertown skin working with live data via websockets.

There are two variants of Belchertown at this time:
* the original by Pat O'Brien -  [(link)](https://github.com/poblabs/weewx-belchertown)
* the fork by 'uajqq' -  [(link)](https://github.com/uajqq/weewx-belchertown-new)

Both variants are installed and configured identically.  The only difference is that the fork is under active development at this time.  The original Belchertown has not been updated in a couple years.

## About Security

> [!IMPORTANT]
> This document assumes there is 'nothing' private in your station web data, meaning that we will not attempt to encrypt the MQTT nor Websockets traffic in these instructions.
>
> We 'highly' recommend securing the ability to write to your MQTT broker with a username/password and 'not' permitting unauthenticated world-write.

## Basic Steps
The typical WeeWX procedures apply for installing and enabling the Belchertown skin, but enabling websockets support has additional steps.  Some use of `sudo` to do privileged tasks is required:

Install Belchertown
* Install weewx
* Install a webserver and integrate it with weewx
* Install the Belchertown skin
* Enable Belchertown for initial testing
* Restart weewx

Reconfigure to enable websockets
* install mosquitto MQTT and its prerequisites
* configure and test the mosquitto broker
* reconfigure weewx and the Belchertown skin to use websockets
* restart weewx
* do a final test

---
## Install Belchertown

### 1. Install weewx
Follow the weewx QuickStart instructions [(link)](https://www.weewx.com/docs/5.2/)

Note - For 'pip'installations remember to activate your python venv before running `weectl`

### 2. Install a webserver and integrate it with WeeWX

* `sudo apt install -y nginx`
* `sudo mkdir /var/www/html/weewx`
* for pip installations:
    * `ln -s /home/YOURUSER/weewx-data/archive /var/www/html/weewx`
    * `sudo chown YOURUSER /var/www/html/weewx`
* for packaged installations:
    * typically the package installer will do the right thing for you

The result is that your weewx web pages will be a `http://YOURHOSTHERE/weewx` by default
    
### 3. Install the Belchertown skin

Run `weectl extension install` and specify the URL of the skin:

* for the original Belchertown:
    * `https://github.com/poblabs/weewx-belchertown/archive/refs/heads/master.zip`
* for the fork:
    * `https://github.com/uajqq/weewx-belchertown-new/archive/refs/heads/master.zip`


### 4. Enable Belchertown for initial testing

Restart weewx via `sudo systemctl restart weewx` and wait your archive period, typically 5 minutes, for weewx to run its reports. Then open the Belchertown skin output URL in your browser.

Initial default setup will write Belchertown skin output to a 'belchertown' (lower case) directory under the top of your weewx HTML document toplevel, such as `http://myhostname/weewx/belchertown` or the like.

>[!NOTE]
>Belchertown is unusual in that it does its magic via javascript which is generated as a weewx report output file. When you change anything Belchertown related in weewx.conf you need to wait until the reports run and the .js file is updated.  You might need to stop/start or reload your browser to make the underlying javascript take effect.

### 5. Restart weewx
Normal weewx procedures apply, typically `sudo systemctl restart weewx`

Check your system logs to ensure weewx works reliably through a couple report cycles.

>[!NOTE]
> Belchertown can throw an error or two the first time it runs on a clean system with a new database.  These clear up the second time the skin runs.

By default if you are using http (not https) you may see errors related to 'Windy' trying to load a map over https. This is not a WeeWX nor Belchertown error.  Browsers typically no longer like web pages that are unencrypted over http 'also' referencing pages that are encrypted using https.  Just ignore those errors for now...


----

## Reconfigure to enable websockets

Now that WeeWX and Belchertown are stable, we enable websockets support.

### 6. install mosquitto MQTT software and other prerequisites

Install the mosquitto software:
````
sudo apt-get install -y mosquitto mosquitto-clients
````

Install the paho-mqtt library

* for pip installations
    * `pip3 install paho-mqtt`

* for packaged installations
    * `sudo apt-get install python3-paho-mqtt`


### 7. configure and test the mosquitto broker


Initial test - pub/sub with no user/pass using localhost

Then set up user/pass/acl and desired acl restrictions and retest pub/sub and acl work as desired

````
TO DO - more details on how
TO DO - how to edit the various files and their permissions
````

### 8. reconfigure weewx and the Belchertown skin to use websockets

Temporary - the configure-websockets-no-encryption document here for the long howto....


### 9. restart weewx
Again, normal weewx procedures apply, typically `sudo systemctl restart weewx`

Check your system logs to ensure weewx works reliably through a couple report cycles.

### 10. do a final test

Now reopen your Belchertown skin output URL and see if it works....
* you should see 'connecting' for a few seconds
* then you should see live data updating every few seconds

If you see connection failed....
````
TO DO - what do they do ?
````

----
## Futures

Add more info on how to encrypt the MQTT traffic and switch to port 8883
* this will need a howto of its own, based on which certs to use and how to get them
* using LE certs might be problematic since they rotate so quickly - how will a server setup keep up ?
* using self-signed certs might be even more difficult
* and how could folks 'securely' run on LAN with access tunneled or proxied back in from Internet ?

----
## (FILES BELOW HERE TO BE EVENTUALLY CUT/PASTED IN ABOVE)
----

## Example files


Output of `locale` - (potentially not necessary):
```
LANG=en_US.UTF-8
LANGUAGE=
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC="en_US.UTF-8"
LC_TIME="en_US.UTF-8"
LC_COLLATE="en_US.UTF-8"
LC_MONETARY="en_US.UTF-8"
LC_MESSAGES="en_US.UTF-8"
LC_PAPER="en_US.UTF-8"
LC_NAME="en_US.UTF-8"
LC_ADDRESS="en_US.UTF-8"
LC_TELEPHONE="en_US.UTF-8"
LC_MEASUREMENT="en_US.UTF-8"
LC_IDENTIFICATION="en_US.UTF-8"
LC_ALL=
```

Mosquitto acl file:
```
topic read $SYS/#

#---------------------
#--- weewx related ---
#---------------------

# user r/w for these topics
user weemqtt
topic weewx/#
topic weetest/#
topic weather/#

#---------------------

# This affects all clients.
pattern write $SYS/broker/connection/%c/state

```
Mosquitto pwfile:
```
weemqtt:$7$101$a14xGSHNx4BHY3vF$OlDvizhNhZYhrzUBH+SKEmLJTgwnwZJqqMTECsMu6Xw5iFfBI8ETKJ+xckOM0qA3SDPC6d1fQKLTCh7yf6CzLQ==
```

Mosquitto pskfile for reference:
```
weemqtt:weepass
```

Mosquitto conf.d/local.conf file:
```
persistence false
allow_anonymous true
password_file /etc/mosquitto/pwfile
acl_file /etc/mosquitto/acl
listener 1883
protocol mqtt
listener 9001
protocol websockets
```

Mosquitto mosquitto.conf:
```
# Place your local configuration in /etc/mosquitto/conf.d/
#
# A full description of the configuration file is at
# /usr/share/doc/mosquitto/examples/mosquitto.conf.example

#pid_file /run/mosquitto/mosquitto.pid

persistence true
persistence_location /var/lib/mosquitto/

log_dest file /var/log/mosquitto/mosquitto.log

include_dir /etc/mosquitto/conf.d

#---- dial up the logging ----
# uncomment some or all of these for more logging
# (they can be 'very' verbose)
#
# log_type debug
# log_type error
# log_type warning
# log_type notice
# log_type information
# log_type subscribe
# log_type unsubscribe
# log_type websockets
# log_type all
```

weewx.conf MQTT section:
```
    [[MQTT]]
        enable = true
        server_url = mqtt://weemqtt:weepass@34.210.111.113:1883/
        topic = weather
        aggregation = aggregate
        binding = loop,archive
        log_success = true
        log_failure = true
```
weewx.conf Belchertown mqtt items:
```
           mqtt_websockets_enabled = 1
           mqtt_websockets_host = 34.210.111.113
           mqtt_websockets_port = 9001
           mqtt_websockets_ssl = 0
           mqtt_websockets_topic = weather/loop
           mqtt_websockets_username = weemqtt
           mqtt_websockets_password = weepass
           disconnect_live_website_visitor = 1800000
           show_last_updated_alert = 0
           last_updated_alert_threshold = 1800
           webpage_autorefresh = 0
```
Permissions - lightly edited to line the columns up:
```
drwxr-xr-x 5 root root           4096 Feb 17 16:08 /etc/mosquitto
drwxr-xr-x 2 root root           4096 Feb 16 12:19 /etc/mosquitto/ca_certificates
-rw-r--r-- 1 root root             73 Apr 15  2024 /etc/mosquitto/ca_certificates/README
-rw-r--r-- 1 root root            230 Sep 18  2023 /etc/mosquitto/aclfile.example
-rw-r--r-- 1 root root            355 Sep 18  2023 /etc/mosquitto/pwfile.example
-rw-r--r-- 1 root root            544 Feb 16 13:46 /etc/mosquitto/mosquitto.conf
drwxr-xr-x 2 root root           4096 Feb 17 08:23 /etc/mosquitto/conf.d
-rw-r--r-- 1 root root            142 Apr 15  2024 /etc/mosquitto/conf.d/README
-rw-r--r-- 1 root root            166 Feb 17 08:23 /etc/mosquitto/conf.d/local.conf
drwxr-xr-x 2 root root           4096 Feb 16 12:19 /etc/mosquitto/certs
-rw-r--r-- 1 root root            130 Apr 15  2024 /etc/mosquitto/certs/README
-rwx------ 1 mosquitto mosquitto   16 Feb 16 15:36 /etc/mosquitto/pskfile
-rw-r--r-- 1 root root             23 Sep 18  2023 /etc/mosquitto/pskfile.example
-rwx------ 1 mosquitto mosquitto  335 Feb 17 16:08 /etc/mosquitto/acl
-rwx------ 1 mosquitto mosquitto  121 Feb 16 15:36 /etc/mosquitto/pwfile
```

weewx extensions
```
$ weectl extension list
Using configuration file /home/ubuntu/weewx-data/weewx.conf
Extension Name    Version   Description
Belchertown       1.7beta2-new-belchertownA clean modern skin with real time streaming updates and interactive charts. Modeled after BelchertownWeather.com
mqtt              0.24vds   Upload weather data to MQTT server.
```

pip modules in the venv
```
$ pip3 list
Package            Version
------------------ -----------
certifi            2026.1.4
charset-normalizer 3.4.4
configobj          5.0.9
ct3                3.4.0.post5
ephem              4.2
idna               3.11
paho-mqtt          2.1.0
pillow             12.1.1
pip                24.0
PyMySQL            1.1.2
pyserial           3.5
pyusb              1.3.1
requests           2.32.5
urllib3            2.6.3
weewx              5.2.0
```