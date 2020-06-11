
# Ubuntu 18.04 + Nginx + Pgadmin4 (Server mode + Reverse Proxy)
![pgadmin](https://github.com/amanjuman/nginx-pgadmin/blob/master/pgadmin.png?raw=true)

### Install required packages:
```
apt install build-essential libssl-dev libffi-dev virtualenv python-pip libpq-dev python-dev
```

### Create directories/permissions:
```
mkdir -p /var/lib/pgadmin4/sessions
mkdir /var/lib/pgadmin4/storage
mkdir /var/log/pgadmin4
chown -R root.root /var/lib/pgadmin4/
chown -R root.root /var/log/pgadmin4/
```

### Enable virtualenv and install pgadmin (at this time 4.22):
```
virtualenv .pgadmin4
cd .pgadmin4/
source bin/activate

wget https://ftp.postgresql.org/pub/pgadmin/pgadmin4/v4.22/pip/pgadmin4-4.22-py3-none-any.whl
pip install pgadmin4-4.22-py3-none-any.whl
```

### Configure server environment:
```
vi nano ~/.pgadmin4/lib/python3.6/site-packages/pgadmin4/config_local.py

---
DEFAULT_SERVER = '0.0.0.0'
DEFAULT_SERVER_PORT = 5050
LOG_FILE = '/var/log/pgadmin4/pgadmin4.log'
SQLITE_PATH = '/var/lib/pgadmin4/pgadmin4.db'
SESSION_DB_PATH = '/var/lib/pgadmin4/sessions'
STORAGE_DIR = '/var/lib/pgadmin4/storage'
SERVER_MODE = True
---
```

### Setup pgadmin4:
```
python lib/python3.6/site-packages/pgadmin4/setup.py

---
NOTE: Configuring authentication for SERVER mode.

Enter the email address and password to use for the initial pgAdmin user account:

Email address: user@domain.com
Password:
Retype password:
pgAdmin 4 - Application Initialisation
======================================
---
```

### Change permission to nginx access:
```
chown -R www-data:www-data /var/lib/pgadmin4/
sudo chown -R www-data:www-data /var/log/pgadmin4/
```

### Create script to start pgadmin:
```
nano /usr/local/bin/pgadmin.sh


#!/bin/bash
. /root/.pgadmin4/bin/activate
# virtualenv is now active.
#
nohup python /root/.pgadmin4/lib/python3.6/site-packages/pgadmin4/pgAdmin4.py &

[SHIFT + zz to save file]
chmod +x /usr/local/bin/pgadmin.sh
```

### Run script:
```
[~/.pgadmin4]: pgadmin.sh
[~/.pgadmin4]: nohup: appending output to 'nohup.out'
```

### Check:
```
[~/.pgadmin4]: netstat -anlp |grep 5050
tcp        0      0 0.0.0.0:5050            0.0.0.0:*               LISTEN      27749/python
```

### Configure nginx domain (attached)
```
server
{
	# Listen
	listen 80;
	listen [::]:80;
	listen 443 ssl http2;
	listen [::]:443 ssl http2;
	
	# Directory & Server Naming
	server_name subdomain.example.com;
	http2_push_preload on;
	
	# HTTP to HTTPS redirection
	if ($scheme != "https")
	{
		return 301 https://$host$request_uri;
	}

	# SSL
	ssl_certificate /etc/letsencrypt/live/subdomain.example.com/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/subdomain.example.com/privkey.pem;
	ssl_trusted_certificate /etc/letsencrypt/live/subdomain.example.com/fullchain.pem;
	
	# Disable Hidden FIle Access Except Lets Encrypt Verification
	location ~ /\.well-known 
	{ 
		allow all;
	}

	# Nginx Logging
	access_log /var/log/nginx/subdomain.example.com-access.log;
	error_log /var/log/nginx/subdomain.example.com-error.log warn;

	# Max Upload Size
	client_max_body_size 100M;
	access_log off;

	location / 
	{
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto https;
		proxy_pass http://127.0.0.1:5050;
	}
}
```
