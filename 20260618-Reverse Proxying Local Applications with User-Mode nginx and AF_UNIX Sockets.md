---
title: "Reverse Proxying Local Applications with User-Mode nginx and AF_UNIX Sockets"
description: "When running multiple local web applications, it is often useful to route requests to different..."
pubDatetime: 2026-06-18T07:11:45.067Z
updatedDate: 2026-07-28T10:58:59.834Z
---

When running multiple local web applications, it is often useful to route requests to different applications based on URL prefixes.

For example:

```text
http://localhost:8080/app1/
```

might need to be reverse proxied to an application running in a separate process.

This is easy to achieve with nginx. However, instead of using a system-wide nginx instance, this setup follows a different approach:

* Run nginx as an unprivileged user
* Use AF_UNIX (Unix domain) sockets instead of TCP ports for backend connections
* Make the URL-to-application mapping obvious from the nginx configuration itself

## Approach

The configuration runs nginx listening on `localhost:8080`.

Requests under `/app1/` are forwarded to a Unix domain socket exposed by the application.

```text
browser
  |
  | http://localhost:8080/app1/
  v
nginx
  |
  | AF_UNIX socket
  v
/home/user/app1/sock.sock
```

Assigning a dedicated TCP port to each application is also possible, but for local development reverse proxies, Unix domain sockets often provide a cleaner layout.

In the configuration, the mapping is explicit:

```nginx
location /app1/ {
    proxy_pass http://unix:/home/user/app1/sock.sock:/;
}
```

From this alone, it is immediately clear which application corresponds to `/app1/`.

## Directory Layout

Create a working directory for nginx:

```text
nginx-local/
├── conf/
│   └── nginx.conf
├── logs/
└── temp/
    ├── client_body/
    ├── proxy/
    ├── fastcgi/
    ├── uwsgi/
    └── scgi/
```

Example setup:

```bash
mkdir -p ./conf
mkdir -p ./logs
mkdir -p ./temp/client_body
mkdir -p ./temp/proxy
mkdir -p ./temp/fastcgi
mkdir -p ./temp/uwsgi
mkdir -p ./temp/scgi
```

## Starting nginx

This nginx instance runs in the foreground:

```bash
nginx -p . -c ./conf/nginx.conf -e ./logs/error.log -g 'daemon off;'
```

The `-p .` option sets the nginx prefix to the current directory.

Relative paths in the configuration, such as `./logs/error.log` or `./temp/proxy`, are resolved relative to this prefix.

Because `-g 'daemon off;'` is specified, nginx does not daemonize and instead runs in the foreground. This is generally more convenient for development and testing. Stop it with `Ctrl-C`.

## Configuration File

`conf/nginx.conf`:

```nginx
worker_processes 1;

error_log ./logs/error.log;
pid       ./nginx.pid;

events {
    worker_connections 1024;
}

http {
    access_log ./logs/access.log;

    client_body_temp_path ./temp/client_body;
    proxy_temp_path       ./temp/proxy;
    fastcgi_temp_path     ./temp/fastcgi;
    uwsgi_temp_path       ./temp/uwsgi;
    scgi_temp_path        ./temp/scgi;

    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        listen 8080;
        server_name localhost;

        location = /app1 {
            return 301 /app1/;
        }

        location /app1/ {
            proxy_pass http://unix:/home/user/app1/sock.sock:/;

            proxy_http_version 1.1;

            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_set_header Upgrade           $http_upgrade;
            proxy_set_header Connection        $connection_upgrade;

            proxy_buffering off;
            proxy_cache off;
            proxy_request_buffering off;

            proxy_read_timeout 3600s;
            proxy_send_timeout 3600s;
        }
    }
}
```

## Important Considerations for Running nginx as an Unprivileged User

When running nginx without root privileges, several points require attention.

First, use a listening port above 1024:

```nginx
listen 8080;
```

Attempting to bind directly to privileged ports such as `80` or `443` as a regular user will fail.

Second, ensure that log files, PID files, and temporary files are written to locations the user can access:

```nginx
error_log ./logs/error.log;
pid       ./nginx.pid;
```

Also explicitly configure temporary directories used by HTTP proxying:

```nginx
client_body_temp_path ./temp/client_body;
proxy_temp_path       ./temp/proxy;
fastcgi_temp_path     ./temp/fastcgi;
uwsgi_temp_path       ./temp/uwsgi;
scgi_temp_path        ./temp/scgi;
```

Without these settings, nginx may attempt to use directories such as `/var/log/nginx/` or `/var/lib/nginx/`, which are typically not writable by unprivileged users.

## Reverse Proxying to AF_UNIX Sockets

To proxy requests to a Unix domain socket, `proxy_pass` can be written as:

```nginx
proxy_pass http://unix:/path/to/sock.sock:/;
```

In this example:

```nginx
proxy_pass http://unix:/home/user/app1/sock.sock:/;
```

the `/app1/` prefix is stripped before forwarding.

Therefore:

```text
http://localhost:8080/app1/foo
```

is forwarded to the backend application as:

```text
/foo
```

If the application needs to receive `/app1/foo` unchanged, the `proxy_pass` URI handling must be adjusted accordingly.

For this setup, nginx uses `/app1/` purely as a routing prefix, while applications receive paths relative to `/`.

## WebSocket Support

To proxy WebSocket connections correctly, HTTP/1.1 and the `Upgrade` / `Connection` headers must be configured:

```nginx
proxy_http_version 1.1;

proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

Since `$connection_upgrade` is not a built-in nginx variable, it is defined via `map` inside the `http` block:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

Always setting `Connection: upgrade` would incorrectly affect normal HTTP requests and SSE connections. This mapping ensures that `upgrade` is only used when an `Upgrade` header is present.

## Server-Sent Events (SSE) / HTTP Event Streams

For SSE and streaming HTTP responses, nginx buffering can prevent events from being delivered immediately.

Disable proxy buffering:

```nginx
proxy_buffering off;
proxy_cache off;
```

Use generous timeouts to allow long-lived connections:

```nginx
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
```

With these settings, standard HTTP traffic, WebSockets, and SSE can all use the same reverse proxy configuration.

## Verification

Validate the configuration:

```bash
nginx -p . -c ./conf/nginx.conf -e ./logs/error.log -t
```

Start nginx:

```bash
nginx -p . -c ./conf/nginx.conf -e ./logs/error.log -g 'daemon off;'
```

Test HTTP:

```bash
curl -i http://localhost:8080/app1/
```

For SSE, disable curl buffering:

```bash
curl -N http://localhost:8080/app1/events
```

For WebSocket testing, tools such as `websocat` are useful:

```bash
websocat ws://localhost:8080/app1/ws
```

## Summary

This approach provides two major advantages.

First, nginx can run entirely in user space.

There is no need to modify system nginx configuration or obtain root privileges. Everything can be managed within a local working directory, making it convenient for development and personal tooling.

Second, using AF_UNIX sockets makes application mappings explicit in the configuration:

```nginx
location /app1/ {
    proxy_pass http://unix:/home/user/app1/sock.sock:/;
}
```

From the nginx configuration alone, it is easy to determine which application serves a given URL prefix.

For managing multiple local web applications behind a single endpoint, the combination of user-mode nginx and AF_UNIX sockets is a practical and maintainable solution.
