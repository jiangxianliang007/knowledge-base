# CKB Node Network Connection TLS Configuration Guide

> This tutorial will guide you through synchronizing data for your deployed CKB node using a domain name (e.g., ckb.example.com), covering DNS setup, CKB configuration adjustments, Nginx reverse proxy setup, and more.

Prerequisites:
- You have deployed a CKB node version 0.200.0+ and it is running normally.
- You own a domain name and can modify its DNS records (e.g., via Cloudflare, Namecheap, Alibaba Cloud, etc.).
- You have a TLS certificate for your domain (you can use a commercial certificate from providers like DigiCert or a free certificate issuance tool like Certbot).
- Ports 80 and 443 are open on your server.
- Nginx is installed with the Stream module enabled, which will be used for traffic distribution.

## Checking if Nginx Has the Stream Module Enabled

Run the following command. If the output includes --with-stream --with-stream_ssl_module --with-stream_ssl_preread_module, the module is enabled:
```
nginx -V 2>&1 | grep -- --with-stream
```

If there’s no output, you’ll need to download the Nginx source code and recompile it with the --with-stream options added to the original compilation parameters:
```
./configure \
  --prefix=/usr/local/nginx \
  --with-stream \
  --with-stream_ssl_module \
  --with-stream_ssl_preread_module
```


## Example nginx.conf Configuration for ckb.example.com

**Notes:**
- Replace ckb.example.com with your own domain.
- Replace 8118 with your CKB node’s network port.
- Replace 443 with the public port you want to expose (ensure it’s open in your firewall/security group).
- 8443 is an internal proxy port and can be customized.
- ssl_certificate and ssl_certificate_key should point to your domain’s TLS certificate files.
```
# Global settings
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

# Events
events {
    worker_connections 768;
}

# ===================
# Stream block (used for TCP/TLS protocol detection and routing)
# ===================
stream {
    log_format stream_log '$remote_addr - $remote_port [$time_local] '
                          'Protocol: $ssl_preread_protocol '
                          'Status: $status '
                          'Bytes_Sent: $bytes_sent '
                          'Bytes_Received: $bytes_received '
                          'Session_Time: $session_time '
                          'Upstream: $upstream_addr';

    # Map TLS version to upstream
    map $ssl_preread_protocol $upstream {
        default      backend_tcp;
        "TLSv1.2"    backend_wss_http;
        "TLSv1.3"    backend_wss_http;
    }

    upstream backend_tcp {
        server 127.0.0.1:8118;
    }

    upstream backend_wss_http {
        server 127.0.0.1:8443;
    }

    server {
        listen 443;
        proxy_pass $upstream;
        ssl_preread on;

        access_log /var/log/nginx/stream_access.log stream_log;
        error_log /var/log/nginx/stream_error.log;
    }
}

# ===================
# HTTP block (for WSS over HTTPS and Web services)
# ===================
http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    client_max_body_size 10m;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    gzip on;

    include /etc/nginx/sites-enabled/*;

    # Server block for WSS proxy (ckb.example.com)
    server {
        listen 8443 ssl;
        server_name ckb.example.com;

        ssl_certificate /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;

        location / {
            proxy_pass http://127.0.0.1:8118;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_read_timeout 3600s;
            proxy_send_timeout 3600s;
        }
    }
}
```

## Traffic Routing
```
TCP Client -> ckb.example.com:443 -> Nginx (stream, ssl_preread) -> backend_tcp -> [ckb node 8118]
WSS Client -> ckb.example.com:443 -> Nginx (stream, ssl_preread) -> backend_wss_http -> 127.0.0.1:8443 -> Nginx (http, TLS decryption, WebSocket) -> [ckb node 8118]
```

## Reload Nginx After Updating nginx.conf
```
sudo nginx -t
sudo nginx -s reload
```

## Add Domain DNS Resolution

Add an A record for the domain ckb.example.com to your CKB node's IP address in your domain registrar.

## Modify ckb.toml to Enable public_addresses

Remove the # in front of public_addresses and enter your domain address in the format: "/dns4/your-domain/tcp/nginx-listening-port"

Edit ckb.toml
```
public_addresses = ["/dns4/ckb.example.com/tcp/443]
```
## Restart CKB
Restart your CKB node after making the changes.

## Verification Steps

1、Fetching Local CKB Node public_addresses

Use the CKB RPC `local_node_info` to retrieve node details
```
echo '{
    "id": 2,
    "jsonrpc": "2.0",
    "method": "local_node_info",
    "params": []
}' | tr -d '\n' | curl -s -H 'content-type: application/json' -d @- http://127.0.0.1:8114  |awk -F ":"  '{print $6}' |awk -F "," '{print $1}'
```

**Sample Output:**

```
"/dns4/ckb.example.com/tcp/443/p2p/QmQDJWySDgJC8eKmdZBMJuYiin5cJUhbYeBjNWvrXRYYUK"
```
2、Initialize a new CKB node with version 0.200.0+

3、Clear the bootnodes and replace it with the public_addresses configured above

Edit ckb.toml
```
bootnodes = [
 "/dns4/ckb.example.com/tcp/443/p2p/QmQDJWySDgJC8eKmdZBMJuYiin5cJUhbYeBjNWvrXRYYUK"
]
```

The wasm light client bootnode needs to have wss added, e.g.
```
bootnodes = [
 "/dns4/ckb.example.com/tcp/443/wss/p2p/QmQDJWySDgJC8eKmdZBMJuYiin5cJUhbYeBjNWvrXRYYUK"
]
```
4、Start the new CKB node and check the logs (it may take some time for the new node to sync, so please be patient). If the node successfully syncs blocks, the configuration is successful.

## [Quick start with docker compose](https://github.com/jiangxianliang007/ckb-network-tls-proxy/blob/main/README.md)
