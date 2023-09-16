# klipper-sv06
this repository contains my klipper configuration files for the sv06 that i torture

I run this on an old medion nuc device using arch linux and have compiled everything manually without the use of kiauh or other tools. 

The stack that I use are klippy, moonraker, mainsail, nginx

Klipper is compiled for my SV06 according to the documentation, using commit 5f990f9 published about 3 weeks ago

## Klippy configuration
The klipper repository has been cloned to /usr/lib/klipper-github, this is where I must go to pull new updates and rebuild the firmware. Below is the systemd service file that starts klippy when i turn on the nuc.

> /etc/systemd/system/multi-user.target.wants/klipper.service

    [Unit]
    Description=3D printer firmware with motion planning on the host
    After=network.target
    
    [Install]
    WantedBy=multi-user.target
    
    [Service]
    Type=simple
    User=klipper
    RemainAfterExit=no
    Environment=PYTHONUNBUFFERED=1
    ExecStart=/usr/bin/python /usr/lib/klipper-github/klippy/klippy.py /var/opt/moonraker/config/klipper.cfg -I /run/klipper/sock -a /var/opt/moonraker/comms/moonraker.sock -l /var/opt/moonraker/logs/klippy.log -a /tmp/klipper_uds
    Restart=always
    RestartSec=10


## Moonraker configuration
The klipper configuration files are moved to the moonraker configuration directory for convenience. I can find them in /var/opt/moonraker. Moonraker runs as its own user, so does klipper.
> /etc/systemd/system/multi-user.target.wants/moonraker.service

    [Unit]
    Description=Moonraker Klipper HTTP Server
    Requires=klipper.service
    After=network.target klipper.service
    
    [Service]
    Type=simple
    User=klipper
    WorkingDirectory=/var/opt/moonraker
    SupplementaryGroups=moonraker-admin
    SyslogIdentifier=moonraker
    RemainAfterExit=yes
    EnvironmentFile=/var/opt/moonraker/systemd/moonraker.env
    ExecStart=/var/lib/klipper/.local/pipx/venvs/moonraker/bin/python -m moonraker $MOONRAKER_ARGS
    Restart=always
    RestartSec=10
    
    [Install]
    WantedBy=multi-user.target


> /var/opt/moonraker/systemd/moonraker.env

    MOONRAKER_ARGS="-v -g -d /var/opt/moonraker -c /var/opt/moonraker/config/moonraker.conf -l /var/opt/moonraker/logs/moonraker.log"

## nginx configuration 

> /etc/nginx/sites-enabled/mainsail

    server {
        listen 80 default_server;
    
        access_log /var/log/nginx/mainsail-access.log;
        error_log /var/log/nginx/mainsail-error.log;
    
        gzip on;
        gzip_vary on;
        gzip_proxied any;
        gzip_proxied expired no-cache no-store private auth;
        gzip_comp_level 4;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        gzip_types text/plain text/css text/xml text/javascript application/javascript application/x-javascript application/json application/xml;
    
        # web_path from mainsail static files
        root /usr/share/webapps/mainsail;
    
        index index.html;
        server_name _;
    
        client_max_body_size 0;
    
        proxy_request_buffering off;
    
        location / {
            try_files $uri $uri/ /index.html;
        }
    
        location = /index.html {
            add_header Cache-Control "no-store, no-cache, must-revalidate";
        }
    
        location /websocket {
            proxy_pass http://apiserver/websocket;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_read_timeout 86400;
        }
    
        location ~ ^/(printer|api|access|machine|server)/ {
            proxy_pass http://apiserver$request_uri;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Scheme $scheme;
        }
    
    }
    upstream apiserver {
        ip_hash;
        server 127.0.0.1:7125;
    }

## other interesting urls for later
https://github.com/rootiest 
https://ellis3dp.com/Print-Tuning-Guide/  
https://www.advanced3dprinting.com/flow-rate-calculator/  
https://www.th3dstudio.com/klipper-abl-mesh-homing-calculator/  
https://teachingtechyt.github.io/calibration.html#esteps
