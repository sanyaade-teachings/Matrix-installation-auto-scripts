README
🚀 Matrix server
An installer has been added to the repository for LL; you can deploy the Matrix server with a single command:

bash <(curl -sSL https://raw.githubusercontent.com/crazy-alert/Matrix/refs/heads/main/installer.sh?timestamp=123)
Also in this repository:

file Readme.element.md- description of the Element Web client settings
File Readme.turn.md- description of TURN server settings for organizing audio and video calls in Matrix (it already works, but you never know...)
Below is the installation procedure without using the automatic installer (not for LL)
📦 Docker stack composition
The stack consists of several containers, which together form a full-fledged Matrix server with a web client, a TURN server for calls, and automatic HTTPS.
caddy A reverse proxy server with automatic SSL certificate generation (Let's Encrypt). Routes traffic to internal services: matrix.${DOMAIN} → synapse, ${DOMAIN} → element-web, admin.${DOMAIN} → synapse-admin.
permissions Auxiliary one-time container (based on alpine). Fixes permissions on the synapse_data volume, setting the owner to 991:991 (the UID of the synapse user in the container). Without this step, Synapse will not be able to write logs, media files, and keys.
synapse The main Matrix server (a Python implementation of Synapse). Stores data in the synapse_data volume and reads configuration from the mounted homeserver.yaml. Dependent on PostgreSQL.
synapse_db A PostgreSQL database used by Synapse to store metadata, rooms, users, etc. Data is stored in the synapse_db_data volume.
coturn A TURN server for organizing audio and video calls via Matrix (when clients cannot connect directly). Configuration is generated on the fly from a template with environment variable substitution.
element-web The Element (formerly Riot) web client, through which users log in to their Matrix account. The element-config.json configuration file is mounted in the container.
synapse-admin An administrative panel for managing users, rooms, and viewing statistics. Accessible via a subdomain (e.g., admin.${DOMAIN}).
Remember and volume
internal– an isolated bridge network through which containers communicate with each other (only Caddyhas access to ports 80/443 to the outside).
caddy_data, caddy_config– volumes for storing certificates and settings Caddy.
synapse_data– volume for media files, logs and keys Synapse.
synapse_db_data– volume for database files PostgreSQL.
It sounds complicated, but with this repository you can actually get it up and running in minutes.
3 steps with tips, everything is described

🔧 Prerequisites
Server (VPS or dedicated) with Ubuntu 20.04+ / Debian 11+ , minimum 1 GB RAM (2 GB recommended).
Docker and Docker Compose installed (usually docker composeincluded with Docker).
A domain name pointing to your server's IP address. You'll need two subdomains:
matrix.ваш-домен.ru– for the Matrix server
(optional) element.ваш-домен.ru– if you want to install the Element web client later
Open ports in the firewall:
80/tcp , 443/tcp – for web interface and clients
3478/udp and 49160-49200/udp – for the TURN server (calls)
(optional) 8448/tcp – for federation, unless you are using delegation via .well-known
Let's get started
0. Updating the system, installing dependencies
apt update && apt install -y git 
1. Clone the repository and switch to it
Create a directory in which we will install, for example /opt/Matrix, go to it and copy this repository into it

mkdir /opt/Matrix &&
cd /opt/Matrix &&
git clone -v  https://github.com/crazy-alert/Matrix.git . 
2. Setup
Run the following command; it will copy the sample configuration file to .envand open it for editing in the editor nano. In the editor, nanothe keyboard shortcuts are: Ctrl+o- Save changes (then press Enter), Ctrl+x- Close.

.envThis is the main server config. Be sure to replace:
DOMAIN=example.org – your domain (replace example.orgwith your domain).
MATRIX_SERVER_NAME=matrix.example.org – this is the direct address of your Synapse server (usually the 'matrix' subdomain is used).
COTURN_EXTERNAL_IP=your_public_ip - substitute the external IP of the server (can be found out using the command hostname -I | awk '{print $1}')
COTURN_INTERNAL_IP=your_internal_ip - internal IP within the network, usually the same as the external one. Command
cp example.env .env &&
nano .env
🔧 Automatic configuration generation ( generate_config.sh) The script generate_config.shcreates final configuration files based on templates and variables from .env.

What it does:

Checks for the existence of the file .envand loads variables.
Generates random secrets ( macaroon, registration shared secret, form secret) if they are not specified, and appends them to .env.
Creates homeserver.yamlfrom a template template.yamlby substituting the server name and password PostgreSQL.
Creates element-config.jsonfrom a template element-config.json.templatefor the Element web client.
Sets permissions to 644 so containers can read files.
Run this script after setting up .env and before starting the Docker stack for the first time.

Command to run:

chmod +x generate_config.sh && ./generate_config.sh
3. 🚀 Server launch
Run in the directory with docker-compose.yml:

docker-compose up -d
In a minute, all containers will be running. Check the logs:

docker compose logs -f
Creating users:
You will be asked to enter:
username (without domain, for example friend)
password
password confirmation
Make me an administrator (answer yes or no)
docker-compose exec synapse register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008
View users:
docker-compose exec synapse_db psql -U synapse -d synapse -c "SELECT name FROM users;"
(All these operations are available via the web when installed synapse-admin, if you haven’t changed anything)

Federation Check
Now the most important thing is to check if your server is visible to others. Use the official Matrix federation tester:
Go to https://federationtester.matrix.org/ Enter your primary domain and click "Go." You should see a green report with no critical errors (Checks - OK, MatchingServerName - OK).

Final check
Try joining a public room, for example #synapse:matrix.org, from your Element client. If the room joins successfully and you see messages, the federation is working perfectly.
Coturn is the most finicky part. If calls aren't working, check the logs and firewall settings (UDP ports 3478, 50000-51000 should be open). Installing Coturn on the host machine (outside of Docker) is often more reliable.
Add rules:

ufw allow 3478/udp
ufw allow 49160:49200/udp
ufw reload
Make sure the rules are added:

ufw status numbered
You should see something similar to this:

Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 80/tcp                     ALLOW IN    Anywhere                   # HTTP
[ 2] 443/tcp                    ALLOW IN    Anywhere                   # HTTPS
[ 3] 8448/tcp                   ALLOW IN    Anywhere                   # Matrix (или другой сервис)
[ 4] 22/tcp                     ALLOW IN    Anywhere                   # ssh
[ 5] 3478/udp                   ALLOW IN    Anywhere
[ 6] 50000:51000/udp            ALLOW IN    Anywhere
[ 7] 80/tcp (v6)                ALLOW IN    Anywhere (v6)              # HTTP
[ 8] 443/tcp (v6)               ALLOW IN    Anywhere (v6)              # HTTPS
[ 9] 8448/tcp (v6)              ALLOW IN    Anywhere (v6)              # Matrix (или другой сервис)
[10] 22/tcp (v6)                ALLOW IN    Anywhere (v6)              # ssh
[11] 3478/udp (v6)              ALLOW IN    Anywhere (v6)
[12] 50000:51000/udp (v6)       ALLOW IN    Anywhere (v6)
View users:

docker exec -it synapse_db psql -U synapse -d synapse -c "SELECT name FROM users;"
To set (change) a password for an existing user in Synapse, run the command:

docker exec -it synapse register_new_matrix_user -c /data/homeserver.yaml -u ИМЯ_ПОЛЬЗОВАТЕЛЯ -p НОВЫЙ_ПАРОЛЬ http://localhost:8008
Addresses:
https://admin.your_server.com - Synapse-admin panel
https://element.your_server.com - Element web client (like web.whatsapp.com or web.telegram.org)
https://federationtester.matrix.org/?server_name=your_server.com - you can check the federation
https://matrix.yourserver.com - should redirect you to matrix.your_server.com_matrix/static/ on the Synapse page
Done! But one thing:
Now anyone can register on your server (in clients or via element-web).
Synapse login management is configured in the homeserver.yaml file. Currently, you have the following settings enabled:

enable_registration: true
enable_registration_without_verification: true
This means that anyone can register (even without email confirmation).
🔧 Registration restriction options:
Disabling registration completely (manual user creation only): The easiest way is to disable registration completely. Then, only administrators will be able to create new accounts via the command register_new_matrix_useror via the API registration_shared_secret(you already have one). Change homeserver.yaml to:

enable_registration: false
# enable_registration_without_verification можно удалить или закомментировать
Save the file and restart Synapse:

docker-compose restart synapse
or

docker compose restart synapse
After this, the registration button in Element Web will disappear, and the attempt to register through the client will be rejected.

Invite-only registration (with tokens) If you want users to be able to register independently, but only via special invitation links, enable token-based registration.

Setting:
Set enable_registration: true (leave as is).
Add parameter:
registration_requires_token: true
Generate invitation tokens. This can be done via the API or a utility register_new_matrix_userwith the [option] option --token. For example, log into the container synapseand run:
docker-compose exec synapse register_new_matrix_user --token=TOKEN_ДЛЯ_ПРИГЛАШЕНИЯ -c /data/homeserver.yaml https://localhost:8008
(You don't have to specify the user; the utility will ask for it separately)

Alternatively, use the client API to bulk generate tokens. Once enabled, registration_requires_tokenyou'll be prompted to enter the token during registration (this field usually appears in the client).
Email domain restrictions (if using email) If you plan to verify email and want to allow registration only with certain addresses, you can set up:

enable_registration: true
enable_registration_without_verification: false  # требовать подтверждения email
registrations_require_3pid:
  - email
allowed_local_3pids:
  - medium: email
pattern: "^.*@ваш-домен\\.ru$"   # регулярка для разрешённых доменов
Don't forget to set up email sending (email parameters in the config) – otherwise confirmation won't work.
