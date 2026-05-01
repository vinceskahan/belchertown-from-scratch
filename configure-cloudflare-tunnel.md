
## Tunnel Belchertown+Websockets to your LAN weewx server

This document shows how to get Belchertown+Websockets tunneled into your LAN via CloudFlare so that you can access realtime weather data both on your LAN and via Internet. 

It assumes you have previously set up a functioning LAN-only Belchertown+Websockets ala 'Belchertown from scratch' [(link)](https://github.com/vinceskahan/belchertown-from-scratch/blob/main/README.md) and that your web configuration expects the Belchertown web pages at http://your.host.name/weewx/belchertown



---
#### 1. Install cloudflared

From the Cloudflare debian installation instructions...
```
sudo mkdir -p --mode=0755 /usr/share/keyrings

curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | \
       sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | \
   sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt-get update && sudo apt-get install cloudflared
```

#### 2. Configure cloudflared
(ref: https://erisa.dev/exposing-a-web-service-with-cloudflare-tunnel/)

##### (a) authorize creating a tunnel on the domain you select

```
cloudflared tunnel login

# Open the resulting link in a browser
#   which creates $HOME/.cloudflared and a cert.pem file therein
```

##### (b) create the tunnel
```
cloudflared tunnel create weewx-mqtt

# This creates a $HOME/.cloudflared/<TUNNEL_ID>.json credentials file.
# You'll need this later in this procedure.
```

##### (c) make a required directory

```
sudo mkdir /etc/cloudflared
```

##### (d) create /etc/cloudflared/config.yml

```
# this creates a locally-managed tunnel with two published applications
# for each ingress service.  The '404' service is a catchall that is required.
# 
# edit the file to contain:
#
    
    tunnel: <TUNNEL_ID>
    credentials-file: /home/pi/.cloudflared/<TUNNEL_ID>.json

    ingress:
        - hostname: www.mydomain.org
            service: http://127.0.0.1:80

        - hostname: mqtt.mydomain.org
            service: ws://localhost:9001
        
        - service: http_status:404
```

##### (e) install the service itself

```

sudo cloudflared service install

```

##### (f) enable cloudflared and start it up

```
sudo systemctl enable cloudflared
sudo systemctl start  cloudflared
```

##### (g) lastly - add DNS records to route to the tunnel for each FQDN

```
cloudflared tunnel route dns weewx-mqtt www.mydomain.org
cloudflared tunnel route dns weewx-mqtt mqtt.mydomain.org

# wait a bit for DNS to propogate, which can take some minutes occasionally
```


#### 3. Verify the cloudflared setup

##### (a) verify the tunnel list

```
pi@raspberrypi:~ $ cloudflared tunnel list
You can obtain more detailed information for each tunnel with `cloudflared tunnel info <name/uuid>`
ID                                   NAME       CREATED              CONNECTIONS
<TUNNEL_ID>                        weewx-mqtt 2026-05-01T03:16:47Z 2xsea01, 1xsea06, 1xsea08
```

##### (b) verify routes for the tunnel

In the Cloudflare web gui, go to `Networking => Tunnels` and verify your setup
You should a `weewx-mqtt` tunnel with one replica and two apps for routes

Click on the `weewx-mqtt` name and then the `Routes` tab on the next page
and verify the routes  You should see:

```
www.mydomain.org   Type 'Published application' with Service 'http://127.0.0.1:80'
mqtt.mydomain.org  Type 'Published application' with Service 'ws://localhost:9001'
```

##### (c) reverify the DNS records propogated and are working

```
sudo apt install host
run 'host www.mydomain.org'
run 'host mqtt.mydomain.org'
```

If they don't resolve initially, wait a bit for DNS to propagate
this might take some minutes to complete to the point where
the `host` commands work

##### (d) check dns records

You should see two records of type `Tunnel` pointing to `weewx-mqtt` (the tunnel name) and `Proxied`.  There should be one for `www` and one for `mqtt`


#### 4. Reconfigure weewx.conf

Your Belchertown section should contain at a minimum:

    mqtt_websockets_enabled = 1
    mqtt_websockets_host = mqtt.mydomain.org
    mqtt_websockets_port = 443
    mqtt_websockets_ssl = 1
    mqtt_websockets_topic = weather/loop

Your MQTT setting should contain:

    [[MQTT]]
        server_url = mqtt://localhost:1883/
        enable = true
        aggregation = aggregate
        binding = loop,archive
        log_success = failure
        log_failure = true
        topic = weather

You can also verify it is working with tools such as MQTT Explorer
or setting log_success = true and consulting the weewx logs

#### 5. Restart weewx

```
sudo systemctl restart weewx

# wait until weewx runs its reports
# which might be as long as your archive period
```

Or optionally run the reports manually
```
# If you have a pip installation, activate your venv
source ~/weewx-venv/bin/activate

# Run the reports manually
weectl report run
```
#### 6. Test everything together end-to-end

Open `https://www.mydomain.org/weewx/belchertown` in your browser
and it should show live updates working

See the 'Belchertown from scratch' pages for what you should see in the browser
for both success and failure scenarios.

At this point your installation is complete

---

### Addendum (optional) - reconfigure nginx so Belchertown pages are at the web root

The following nginx sites-enabled file configures the belchertown web pages as your web root
so https://hostname/ displays the belchertown page.  If you use this configuration, be sure to restart nginx so it takes effect.

(ref: https://www.woodlands-weather.co.za/installation-guide/WeeWX_Installation_Guide)

```
server {
     listen 80;
     server_name _; # catches all IPs
     root /var/www/html;
     index index.html index.htm;

     # Main location for the site
     location / {
         try_files $uri $uri/ =404;
     }

     # Specific location for weewx + belchertown
     location /weewx/ {
         alias /var/www/html/weewx/;
         try_files $uri $uri/ =404;
         index index.html;
         autoindex off;
     }

     # Redirect root directly to Belchertown
     location = / {
         return 301 /weewx/belchertown/;
     }

     client_max_body_size 10M;
}
```
----
