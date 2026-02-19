
# Belchertown websockets from scratch
The intent of this document is to do a step-by-step walkthrough of how to get the weewx Belchertown skin working with live data via websockets.

This is documented in two pieces, first the basic weewx and nginx installation and configuration, and then a separate document with how to enable and test websockets and the Belchertown skin end-to-end.


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
## Install WeeWX and the Belchertown skin

### 1. Install weewx
Follow the weewx QuickStart instructions [(link)](https://www.weewx.com/docs/5.2/)

Note - For 'pip'installations remember to activate your python venv before running `weectl`

### 2. Install a webserver and integrate it with WeeWX

  * `sudo apt install -y nginx`
  * `sudo mkdir /var/www/html`

  * for pip installations:
    * `sudo mkdir -p /var/www/html/weewx`
    * `sudo chown pi /var/www/html/weewx`
    * `sudo ln -s /var/www/html/weewx /home/YOURUSER/weewx-data/public_html`
* for packaged installations:
    * typically the package installer will do the right thing for you

The result is that your weewx web pages will be at `http://YOURHOSTHERE/weewx` by default
    
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

Please see [LINK](configure-websockets-no-encryption.md) for the long howto....


----
## Futures

Add more info on how to encrypt the MQTT traffic and switch to port 8883
* this will need a howto of its own, based on which certs to use and how to get them
* using LE certs might be problematic since they rotate so quickly - how will a server setup keep up ?
* using self-signed certs might be even more difficult
* and how could folks 'securely' run on LAN with access tunneled or proxied back in from Internet ?

----

### Detailed WeeWX configuration info

weewx extensions
```
$ weectl extension list
Using configuration file /home/ubuntu/weewx-data/weewx.conf
Extension Name    Version   Description
Belchertown       1.7beta2-new-belchertownA clean modern skin with real time streaming updates and interactive charts. Modeled after BelchertownWeather.com
mqtt              0.24   Upload weather data to MQTT server.
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
----
### Credits

Thanks to Gary Hammer for helping me battle through this....