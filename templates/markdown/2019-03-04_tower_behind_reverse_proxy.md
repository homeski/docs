## Ansible Tower behind a reverse proxy

When putting Ansible Tower behind a reverse proxy for whatever reason, it's easy to be surprised when live events (auto-refresh of job templates output for example) in the web interface doesn't work anymore. If your reverse proxy is handled by [NGINX](https://www.nginx.com/resources/wiki/) or [Apache](https://httpd.apache.org/), they have to be specifically configured to pass the [Websocket protocol](http://www.websocket.org/) as well. The Ansible documentation mentions it's use of WebSockets [briefly](https://docs.ansible.com/ansible-tower/latest/html/administration/troubleshooting.html#websockets-port-for-live-events-not-working).

### Errors

Tower doesn't throw show any errors in the UI but if a browser developer console is opened, errors like the following can be seen repeatedly:

```
WebSocket connection to '<URL>' failed:
WebSocket is closed before the connection is established.
```

or

```
WebSocket connection to 'wss://tower.escwq.com/websocket/' failed:
Error during WebSocket handshake: Unexpected response code: 504
```

**If you're seeing the first error, you may try setting your [`REMOTE_HOST_HEADERS`](https://docs.ansible.com/ansible-tower/latest/html/administration/proxy-support.html#reverse-proxy) setting first.**

<img width="100%" src="images/tower-errors.png"/>

### NGINX

A sample NGINX reverse proxy config for Tower:

```
server {
  listen 443;
  server_name tower.example.com;

  location / {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_redirect off;
    proxy_pass https://tower.example.com/;
  }

  location /websocket/ {
    proxy_http_version 1.1;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_pass https://tower.example.com/websocket/;
  }

  ssl on;
  ssl_certificate /etc/tower/tower.cert;
  ssl_certificate_key /etc/tower/tower.key;
}
```

### Apache

A sample Apache reverse proxy config for Tower:

```
<VirtualHost *:80>
  ServerName tower.example.com

  RewriteEngine On
  RewriteCond %{HTTP:Upgrade} =websocket [NC]
  RewriteRule /(.*)           ws://tower.host.name:80/$1 [P,L]
  RewriteCond %{HTTP:Upgrade} !=websocket [NC]
  RewriteRule /(.*)           http://tower.host.name:80/$1 [P,L]

  ProxyPassReverse / http://localhost:80/
</VirtualHost>
```

Restart the server after any config file changes and live events should work!
