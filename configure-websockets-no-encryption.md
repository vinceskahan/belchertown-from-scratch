# Configuring websockets without encryption


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
    

    If the locale isn't set properly, consult your os for how to reconfigure your system locale.
    (TO DO - debian instructions)

## 2. Set up MQTT read/write access control:

### Set up the /etc/mosquitto/aclfile 

This example sets up read/write access for a 'weemqtt' user to a number of possible MQTT topics.

    topic read $SYS/#

    #--- weewx related ---
    # user r/w for these topics
    user weemqtt
    topic weewx/#
    topic weetest/#
    topic weather/#

    # This affects all clients.
    pattern write $SYS/broker/connection/%c/state

    
### Set up the /etc/mosquitto/pwfile

This file contains the example MQTT read/write user and its encrypted password.  This example follows a multi-step procedure so you have a cleartext file (appropriately secured) with the user:pass so it's not forgotten or misplaced.
 
First create a cleartext /etc/mosquitto/pskfile containing the example read/write user:password for reference:

    weemqtt:weepass

Next generate a `/etc/mosquitto/pwfile` which will contain the encrypted password rather than a cleartext one. You'll have to `sudo bash` to open a bash shell for these steps.


    cp /etc/mosquitto/pskfile /etc/mosquitto/pwfile
    mosquitto_passwd -U /etc/mosquitto/pwfile

    # (ignore complaint about ownership and permissions)

    # the output should look something like:
    weemqtt:$7$101$a14xGSHNx4BHY3vF$OlDvizhNhZYhrzUBH+SKEmLJTgwnwZJqqMTECsMu6Xw5iFfBI8ETKJ+xckOM0qA3SDPC6d1fQKLTCh7yf6CzLQ==

    chmod 700 /etc/mosquitto/pskfile /etc/mosquitto/pwfile
    chown mosquitto:mosquitto /etc/mosquitto/pskfile /etc/mosquitto/pwfile


### Set up the mosquitto config file

Create `/etc/mosquitto/conf.d/local.conf` containing the following.  Again you'll need to be in a root shell to do this.


    persistence false
    allow_anonymous true
    password_file /etc/mosquitto/pwfile
    acl_file /etc/mosquitto/acl
    listener 1883
    protocol mqtt
    listener 9001
    protocol websockets


### Enable logging via mosquitto.conf as needed

If you want to log verbosely to `/var/log/mosquitto/mosquitto.log`, add and uncomment the appropriate liness below to your /`etc/mosquitto/mosquitto.conf` file.  Again root is needed to do this.

Mosquitto mosquitto.conf:

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


### Verify your permissions

Mosquitto is 'very' sensitive to permissions, and we want to make sure your acl and usernames/passwords are appropriately secured. Run `find /etc/mosquitto | exec ls -lad {} \;` and examine the output.

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

### Restart mosquitto and retest

    sudo systemctl restart mosquitto
    # verify it started ok and is listening on 1883 and 9001
    # retest sub fails no user/pass
    # retest pub fails no user/pass
    # retest no world read
    # retest sub with user/pass
    # retest pub with user/pass
  
  At this point you can now reconfigure weewx.conf to use websockets
