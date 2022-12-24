# UNGINX - non-NGINX webserver using NGINX certbot plugin for certificate renewal

The NGINX certbot plugin is easily installed, which is the main benefit of this technique.
The NGINX TLS config (/etc/nginx/sites-enabled/default) can be set to a separate unused open 
port, anything different than our actual web server that we'll run via docker compose in
this example.

## docker-compose to create the actual TLS webservers at the microservice level, no TLS termination, no gap in the tunnel

Running the real web services in docker but having NGINX and certbot at the host level,
we can keep our `docker-compose.yml` file in `/root/` and then do the following:

```
08 */12 * * * sleep $(echo $RANDOM | cut -c2,3) && /usr/bin/certbot --non-interactive --non-interactive --nginx -d YOURDOMAINHERE -d ANOTHERDOMAINHERE && /usr/bin/pkill nginx && sleep 63 && cd ~ && /usr/bin/docker-compose restart

```

The example crontab includes a `pkill nginx` if the renewal is successful, as the nginx process gets started by certbot
and is only used for the purpose of certificate nenewal.

The reason we are doing a docker-compose restart is because the certificate is then mounted into the containers that need it:

```
version: '3'
services:
  morpho:
    image: localhost:5000/morpho-web:lxo
    container_name: morphodefault
    restart: unless-stopped
    ports:
      - "443:443"
    volumes:
      - /var/www/html/:/app/static/
      - /etc/letsencrypt/live/$example.com/privkey.pem:/app/privkey.pem
      - /etc/letsencrypt/live/$example.com/fullchain.pem:/app/cert.pem
    logging:
        options:
            max-size: "10m"


```

The section with `$example.com` would be the primary domain name used in the certbot renewal.

Here is a manual interactive example. 

```
certbot --nginx -d example.com -d example2.com
```

Replace `$example.com` here with the DNS A record pointed at the server.

The example references morpho-web, see https://github.com/jpegleg/morpho-web. Morpho-web runs an actix web server in a docker container.
We can the mount the certificates to the container in the docker-compose.yml.

## impact to docker-restart

Because after a successful certificate renewal the (actix) container is restarted, there could be impacts from that restart.
To avoid down time, it is best to have a second server in a different geographic region that also has DNS A records for it,
and ideally a network solution "above" both that publishes A records for which ever is up and online. And if desired, the 
DNS/GSLB system might even sync renewal timers and point traffic over to the other region when a region does a renewal. It
can accomplish this because when the NGINX service comes online, that NGINX PORT returning a 200 OK to a GSLB health check 
is a sign that a renewal is in progress. NGINX needs public traffic to come back to it for a moment during the renewal process
to confirm DNS identity, but after that, we don't need to run it. So we need A records to point to the node with the NGINX
up, then when NGINX is pkilled by the crontab, the GSLB knows that the "drop" means it is time to publish the A record for
the other region only. This will allow the cron to sleep for 63 seconds, enough time for the DNS A record to be republished
and TTL resolved, before the cron does the container restart.

If this type of port event logic is used, then the renewals must not overlap! Ensure the crontab execution times do not align.
Techniques such as changing which hours of the day are used or which days of the week are used, can be set to avoid overlap in
renewals.
