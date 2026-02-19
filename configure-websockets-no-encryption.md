# Configure websockets without encryption

Now that we know mosquitto itself is up and running correctly without any access control, we can move on to enabling mosquitto access control and reconfiguring WeeWX and the Belchertown skin for websockets.

## 1. Check your locale

Run the 'locale' command and make sure you have a valid locale set.  For a US English user it should return the following:

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
    

If the locale isn't set properly, consult your os documentation for how to reconfigure your system locale.  For a raspberry pi you can run `sudo raspi-config` and pick the Localization => Locale option in the menu.

## 2. Set up MQTT read/write access control:

You'll have to `sudo bash` to open a bash shell for these steps.

### Create the /etc/mosquitto/aclfile 

This example sets up read/write access for a 'weemqtt' user to a number of possible MQTT topics.

* create the file

        topic read $SYS/#

        #--- weewx related ---
        # user r/w for these topics
        user weemqtt
        topic weewx/#
        topic weetest/#
        topic weather/#

        # This affects all clients.
        pattern write $SYS/broker/connection/%c/state

* and set its permissions correctly

        chown mosquitto:mosquitto /etc/mosquitto/aclfile
        chmod 0700 /etc/mosquitto/aclfile


### Create Mosquitto password files

This example follows a multi-step procedure so you have a cleartext file (appropriately secured) with the user:pass so it's not forgotten or misplaced.
 
* First create a cleartext /etc/mosquitto/pskfile containing the example read/write user:password for reference:

        weemqtt:weepass

* Next generate a `/etc/mosquitto/pwfile` which will contain the encrypted password rather than a cleartext one. 


        # copy the reference file to the final destination
        cp /etc/mosquitto/pskfile /etc/mosquitto/pwfile

        # convert the pwfile to encrypt the password therein
        mosquitto_passwd -U /etc/mosquitto/pwfile

        # ignore the complaint about ownership and permissions
        # the output should look something like:
        weemqtt:$7$101$a14xGSHNx4BHY3vF$OlDvizhNhZYhrzUBH+SKEmLJTgwnwZJqqMTECsMu6Xw5iFfBI8ETKJ+xckOM0qA3SDPC6d1fQKLTCh7yf6CzLQ==

* Last - set their permissions correctly

        chmod 700 /etc/mosquitto/pskfile /etc/mosquitto/pwfile
        chown mosquitto:mosquitto /etc/mosquitto/pskfile /etc/mosquitto/pwfile


### Create the mosquitto site configuration file

This file configures the mosquitto broker to use the aclfile (access control) and pwfile (user authorization), and to listen on the appropriate network ports

* create `/etc/mosquitto/conf.d/local.conf` containing the following:


        persistence false
        allow_anonymous true
        password_file /etc/mosquitto/pwfile
        acl_file /etc/mosquitto/acl
        listener 1883
        protocol mqtt
        listener 9001
        protocol websockets


### (optional) Enable logging via mosquitto.conf as needed

* If you want to (at least temporarily) increase the verbosity of the mosquitto logs, append the following to /etc/mosquitto/mosquitto.conf and uncomment the desired line(s).  This is generally only helpful during initial setup and checkout.  In usual operation the weewx logging should suffice.


        #---- append to /etc/mosquitto/mosquitto.conf to dial up the logging ----
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


## 3. Verify your permissions

Mosquitto is 'very' sensitive to permissions, and it is necessary to validate your acl and usernames/passwords are appropriately secured. Run `find /etc/mosquitto | exec ls -lad {} \;` and examine the output.

The command output should look like the following (lightly edited to line the columns up):

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

Mosquitto will fail to start if permissions are incorrect on the aclfile and pwfile.  We lock down the pskfile because it is an unused hint file for reference so you can find what the authorized user's encrypted password in the pwfile maps to.

## 4. Restart mosquitto and retest

* Restart mosquitto

        # sudo systemctl restart mosquitto

* Test 1 - verify the configured mosquitto is running

        # sudo tail /var/log/mosquitto/mosquitto.log

        1771522864: mosquitto version 2.0.21 starting
        1771522864: Config loaded from /etc/mosquitto/mosquitto.conf.
        1771522864: Opening ipv4 listen socket on port 1883.
        1771522864: Opening ipv6 listen socket on port 1883.
        1771522864: Opening websockets listen socket on port 9001.
        1771522864: mosquitto version 2.0.21 running


* Test 2 - verify anonymous posting to random topic fails

        # mosquitto_pub -h localhost -t testing -m 123

        # tail /var/log/mosquitto/mosquitto.log

        1771523309: New connection from ::1:52338 on port 1883.
        1771523309: New client connected from ::1:52338 as auto-18063815-7B6B-4A95-3C91-4CF687F04CBA (p2, c1, k60).
        1771523309: No will message specified.
        1771523309: Sending CONNACK to auto-18063815-7B6B-4A95-3C91-4CF687F04CBA (0, 0)
        1771523309: Denied PUBLISH from auto-18063815-7B6B-4A95-3C91-4CF687F04CBA (d0, q0, r0, m0, 'testing', ... (3 bytes))
        1771523309: Received DISCONNECT from auto-18063815-7B6B-4A95-3C91-4CF687F04CBA
        1771523309: Client auto-18063815-7B6B-4A95-3C91-4CF687F04CBA disconnected.


* Test 3 - verify anonymous posting to weather topic in the aclfile fails

        # mosquitto_pub -h localhost -t weetest/testing -m 123

        # tail /var/log/mosquitto/mosquitto.log
        1771523141: New connection from ::1:38746 on port 1883.
        1771523141: New client connected from ::1:38746 as auto-43167224-00A5-25C2-F261-E2BAF47DE530 (p2, c1, k60).
        1771523141: No will message specified.
        1771523141: Sending CONNACK to auto-43167224-00A5-25C2-F261-E2BAF47DE530 (0, 0)
        1771523141: Denied PUBLISH from auto-43167224-00A5-25C2-F261-E2BAF47DE530 (d0, q0, r0, m0, 'weetest/testing', ... (3 bytes))
        1771523141: Received DISCONNECT from auto-43167224-00A5-25C2-F261-E2BAF47DE530
        1771523141: Client auto-43167224-00A5-25C2-F261-E2BAF47DE530 disconnected.


* Test 4 - verify posting to topic in the aclfile works if user/pass is specified

        # mosquitto_pub -h localhost -t weetest/testing -u weemqtt -P weepass -m testing123
        
        # tail /var/log/mosquitto/mosquitto.log
        1771523540: New connection from ::1:36396 on port 1883.
        1771523540: New client connected from ::1:36396 as auto-FA93CAAA-CDB8-6DCE-B622-772CF768D0EA (p2, c1, k60, u'weemqtt').
        1771523540: No will message specified.
        1771523540: Sending CONNACK to auto-FA93CAAA-CDB8-6DCE-B622-772CF768D0EA (0, 0)
        1771523540: Received PUBLISH from auto-FA93CAAA-CDB8-6DCE-B622-772CF768D0EA (d0, q0, r0, m0, 'weetest/testing', ... (10 bytes))
        1771523540: Received DISCONNECT from auto-FA93CAAA-CDB8-6DCE-B622-772CF768D0EA
        1771523540: Client auto-FA93CAAA-CDB8-6DCE-B622-772CF768D0EA disconnected.

  
  At this point you can now reconfigure weewx.conf to use websockets


## 5. Install the weewx-mqtt extension

This example uses Matthew's original weewx-mqtt extension, but other options are available.  See the WeeWX wiki for alternate extensions.

    weectl extension install https://github.com/matthewwall/weewx-mqtt/archive/refs/heads/master.zip

This extension comes set 'enabled' but unconfigured by default, so weewx will fail if you restart it immediately after installing the extension.  We configure it next below.

## 6. Reconfigure WeeWX to use websockets

>[!CAUTION]
> Substitute in your ip address or fully-qualified-domain-name or hostname for nnn.nnn.nnn.nnn in the following section.  It is 'critically' important these two match. If you use a name it must resolve successfully on all clients you want to be able to access your site via websockets.

Edit the weewx.conf MQTT section to match your URLs so that weewx publishes to the mosquitto broker you just configured and restarted.

```
    [[MQTT]]
        enable = true
        server_url = mqtt://weemqtt:weepass@nnn.nnn.nnn.nnn:1883/
        topic = weather
        aggregation = aggregate
        binding = loop,archive
        log_success = false        # set true initially to see publishing succeed via your weewx logs
        log_failure = true
```

Edit the Belchertown section of weewx.conf mqtt-related settings:

```
           mqtt_websockets_enabled = 1
           mqtt_websockets_host = nnn.nnn.nnn.nnn
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

## 7. Restart weewx and check for errors

Finally restart weewx and check your weewx logs for errors.  If you have `log_success = true` above (recommended at least initially) you should see weewx publishing to your broker.
    
* If you have rsyslog configured, simply tail the weewxd log

```
    # tail /var/log/weewx/weewxd.log
    2026-02-19T10:06:31.715266-08:00 raspberrypi weewxd[2452]: INFO weewx.engine: Starting main packet loop.
    2026-02-19T10:06:34.092875-08:00 raspberrypi weewxd[2452]: INFO user.mqtt: client established for mqtt://weemqtt:xxx@192.168.1.164:1883/
    2026-02-19T10:06:34.093807-08:00 raspberrypi weewxd[2452]: INFO weewx.restx: MQTT: Published record 2026-02-19 10:06:34 PST (1771524394)
    2026-02-19T10:06:36.588580-08:00 raspberrypi weewxd[2452]: INFO weewx.restx: MQTT: Published record 2026-02-19 10:06:37 PST (1771524397)
```

* The systemd `journalctl` comand can also be used

    ```

    # journalctl -u weewx.service -n 40

    Feb 19 10:06:31 raspberrypi weewxd[2452]: INFO __main__: Starting up weewx version 5.2.0
    Feb 19 10:06:31 raspberrypi weewxd[2452]: INFO weewx.engine: Clock error is -0.13 seconds (positive is fast)
    Feb 19 10:06:31 raspberrypi weewxd[2452]: INFO weewx.engine: Using binding 'wx_binding' to database 'weewx.sdb'
    Feb 19 10:06:31 raspberrypi weewxd[2452]: INFO weewx.manager: Starting backfill of daily summaries
    Feb 19 10:06:31 raspberrypi weewxd[2452]: INFO weewx.manager: Daily summaries up to date
    Feb 19 10:06:31 raspberrypi weewxd[2452]: INFO weewx.engine: Starting main packet loop.
    Feb 19 10:06:34 raspberrypi weewxd[2452]: INFO user.mqtt: client established for mqtt://weemqtt:xxx@192.168.1.164:1883/
    Feb 19 10:06:34 raspberrypi weewxd[2452]: INFO weewx.restx: MQTT: Published record 2026-02-19 10:06:34 PST (1771524394)
    Feb 19 10:06:36 raspberrypi weewxd[2452]: INFO weewx.restx: MQTT: Published record 2026-02-19 10:06:37 PST (1771524397)
    Feb 19 10:06:39 raspberrypi weewxd[2452]: INFO weewx.restx: MQTT: Published record 2026-02-19 10:06:39 PST (1771524399)
    Feb 19 10:06:41 raspberrypi weewxd[2452]: INFO weewx.restx: MQTT: Published record 2026-02-19 10:06:42 PST (1771524402)
    Feb 19 10:06:44 raspberrypi weewxd[2452]: INFO weewx.restx: MQTT: Published record 2026-02-19 10:06:44 PST (1771524404)
    ```

## 8. Try it out !!!!

Typically a system reboot makes this process a little easier. Be sure to wait 5-10 minutes to let things stabilize and have WeeWX run the Belchertown report which generates the underlying javascript that makes the realtime updates subscription work under the hood.

* `sudo systemctl reboot`
* (want 5-10 minutes)
* open up a browser and navigate to `http://x.x.x.x/weewx/belchertown`

>[!TIP]
> Patience is needed here.  It takes a few seconds to start showing realtime updates.  If it is failing for some reason it might take 20 seconds to see the Failed message
> 
### Expected screens in the browser

* You should see a 'connecting' message - [image](connecting-screen.png)
* When it works, you will see the following: [image](success-screen.png)
* If it fails, you will see this: [image](failure-screen.png)

### It failed - now what ?
If you see a failure:
* verify the 'Published record' messages are appearing in your WeeWX log
* verify there are no permission denied or the like in your mosquitto log
* if both look ok:
  * reboot the system
  * wait 10 minutes
  * fully close your browser
  * open your browser and try again

For some os and browsers, MacOS Tahoe 26.2 and Safari for example, initial attempts at seeing the Belchertown realtime updates from a raspberry pi on the same LAN tend to fail until you take the reboot, wait, and try again.  Again - patience and persistence is sometimes needed.