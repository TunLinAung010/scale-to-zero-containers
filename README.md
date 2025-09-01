# Scale to Zero containers management 

Hi everone, i would like to share you that i deploy scale to zero containers management using lightweight tools ( Salbier ).

# Deploy Sablier on docker swarm
sablier-stack.yml
```
version: "3.8"

services:
  sablier:
    image: sablierapp/sablier:1.9.0
    networks:
      - sablier-net
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: ["start", "--provider.name=docker_swarm"]

networks:
  sablier-net:
    external: true

```
# Deploy Portainer services with Sablier label

```
    version: '3.8'

    services:
      agent:
        image: portainer/agent
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          - /var/lib/docker/volumes:/var/lib/docker/volumes
        networks:
          - sablier-net
        deploy:
          mode: global
          placement:
            constraints: [node.platform.os == linux]

      portainer:
        image: portainer/portainer
        command: -H tcp://tasks.agent:9001 --tlsskipverify
        ports:
          - 9000:9000
          - 8000:8000 #changeme
        networks:
          - sablier-net
        deploy:
          mode: replicated
          replicas: 0
          placement:
            constraints:
              - node.role == manager
        labels:
          - sablier.enable=true
          - sablier.group=whoami
        volumes:
          - /home/r00t/cluster-data/containers-data/portainer:/data #changeme if you don't use this file structure


    networks:
      sablier-net:
        external: true

```

To manage Portainer with Scale to Zero when this service inactive after 15 minutes, portainer.conf 

```
#Rerect HTTP to HTTPS (optional but recommended)
server {
    listen 80;
    server_name portainer.softlinkmm.com;
    return 301 https://$host$request_uri;
}

# HTTPS server block
server {
    listen 443 ssl;
    server_name portainer.softlinkmm.com;

    # Self-signed SSL cert and key
    ssl_certificate     /etc/nginx/ssl/certs/softlinkmm.com.crt;
    ssl_certificate_key /etc/nginx/ssl/certs/softlinkmm.com.key;

    # Recommended SSL settings
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    js_import conf.d/sablier.js;
    resolver 127.0.0.11 valid=10s ipv6=off;
    subrequest_output_buffer_size 32k;

    set $sablierUrl /sablier;
    set $sablierSessionDuration 15m;

    location /sablier/ {
        internal;
        proxy_method GET;
        proxy_pass http://sablier:10000/;
    }

    location @portainer {
        set $portainer_server portainer_portainer;
        proxy_pass http://$portainer_server:9000;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        set $sablierDynamicShowDetails true;
        set $sablierDynamicRefreshFrequency 5s;
        set $sablierNginxInternalRedirect @portainer;
        set $sablierNames portainer_portainer;
        set $sablierDynamicName "Dynamic Portainer";
        set $sablierDynamicTheme hacker-terminal;
        js_content sablier.call;
    }
}

```

