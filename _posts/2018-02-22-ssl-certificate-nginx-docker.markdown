---
layout: post
title:  "Setting up an SSL certificate for Nginx in a Docker container"
date:   2018-02-22 17:03:00 +0300
categories: devops
---

Having an SSL certificate is a crucial requirement for Web sites not only because it adds to their
security but, mostly, because it makes them more trustful by their visitors, when they see a green lock in the browser address meaning their connection is secure.

According to this [article](https://www.wired.com/2017/01/half-web-now-encrypted-makes-everyone-safer/), half Web sites still don't put SSL encryption in use. The reasons for that may be various: a lazy site administrator, SSL certificates cost money, and so on.

This essay is about how to set up an SSL certificate in an easy way and free of charge. And because more and more people begin to use Docker in production, we are going to slightly complicate this task by doing it for Nginx running in a Docker container.

Let's start with Dockerfile for our Nginx image:
{% highlight bash %}
FROM nginx:1.13.1

RUN apt-get update && apt-get install -y \
    openssl \
    cron \
    procps \
    certbot

# Cron job for the SSL certificate renewal
RUN set -ex \
    && crontab -l | { cat; echo "59 23 * * * certbot renew --standalone --pre-hook 'nginx -s stop' --post-hook 'nginx -c /etc/nginx/nginx.conf'"; } | crontab -

# Copy nginx config files
COPY nginx.conf /etc/nginx/nginx.conf
COPY ssl-params.conf /etc/nginx/
COPY ./sites-enabled/app.conf /etc/nginx/conf.d/

# Entrypoint
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh

CMD ["/entrypoint.sh"]
{% endhighlight %}
The content of Dockerfile is self-explanatory. We will only mention [Let's Encrypt's cerbot](https://certbot.eff.org/) utility by means of which we will obtain our SSL certificate.

Now, let's take a look at Nginx's container's entrypoint:
{% highlight bash %}
#!/usr/bin/env bash

domain_opts=

openssl dhparam -out /etc/ssl/certs/dhparam.pem 512

/usr/sbin/cron

# Request a certificate if none was installed before
if [[ ! $(find /etc/letsencrypt/live -name '*.pem') ]]; then

    if [ -z "$CERTBOT_DOMAINS" ]; then
        CERTBOT_DOMAINS=yourdomain.com
    fi

    for d in $CERTBOT_DOMAINS; do
        domain_opts="$domain_opts -d $d"
    done

    certbot certonly -n --standalone $domain_opts --agree-tos -m admin@yourdomain.com
fi

nginx -c /etc/nginx/nginx.conf

# /dev/stdout and dev/stderror references to access.log and error.log correspondingly
tail -q -s 10 -f /var/log/nginx/access.log /var/log/nginx/error.log
{% endhighlight %}

The trick is we don't run the Nginx process in foreground. Since we have to re-new our certificate in some time, the Nginx process will be stopped by the cerbot utility - running by the cron scheduler once a day - as soon as our certificate is about for renewal.

And if you stop a container's foreground process the container will terminate its execution. To avoid that, we are using tail command as a workaround for our foreground process.

Example of Nginx configuration for our Web site:

{% highlight nginx %}
# Force using of HTTPS protocol
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_name _;
	return 301 https://$host$request_uri;
}

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;

    add_header Access-Control-Allow-Origin *;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    charset       utf-8;

    set $web_root   /var/www/html/public;
    root $web_root;

    location / {
        try_files $uri /index.php?$query_string;
    }

    location ~ [^/]\.php(/|$) {
        fastcgi_param SCRIPT_FILENAME $web_root/index.php;
        include fastcgi_params;
        fastcgi_pass            php:9000;
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        fastcgi_index           index.php;
    }
}
{% endhighlight %}

The only thing left to do is to start our container, this might look like this:

{% highlight bash %}
start_nginx() {
    local remote_host="$1"
    local image="$2"

    indicator_start 'Starting Nginx service'

    (
    ssh "$remote_host" "docker run -d -p 80:80 -p 443:443 --name=$NGINX_CONT_NAME \
        -e CERTBOT_DOMAINS='yourdomain.com' \
        -v /etc/letsencrypt:/etc/letsencrypt \
        --link=$PHP_CONT_NAME --volumes-from=$PHP_CONT_NAME:ro $image"
    ) > /dev/null

    indicator_stop
}
{% endhighlight %}

And that's it! Your site is up and running with the SSL certificate you have installed.