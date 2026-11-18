# Docker on Azure: build your own Nextcloud file storage

<em>Note from Aurélien: tutorial composited from different web sources, I used GPT to format them together in a Markdown file</em>

By the end of this lab, you’ll upload and download a file through Nextcloud at your tutor-assigned HTTPS address.

## 1. Create a fresh Ubuntu VM

> **Lab capacity.** The VM is small and it will not handle more than one user and a couple of small files.

1. Open **Tutorial 1** from your course materials and start with its Azure VM creation steps. In **Azure for Students → Free Services → Compute**, create a fresh Ubuntu VM using `Standard_B2ats_v2` and give it a public IPv4 address.
2. **If you deleted your previous VM**, follow the same VM creation steps in Tutorial 1. As soon as the new public IPv4 address appears on the VM’s **Overview** page, **send it to Aurelien ASAP** so he can update your DNS record. If you still have your previous VM, leave it in place and send Aurelien the new VM’s public IPv4 address too.
3. Follow Tutorial 1’s SSH connection steps to sign in to the new VM. Stop Tutorial 1 there. Skip its Apache, PHP, MariaDB and Let’s Encrypt setup.
4. Your assigned address will have the form **`subdomain.fifecomptech.net`**. Replace `subdomain` with the name Aurelien assigns you wherever it appears below. For example, if your assigned name is `alex`, use `alex.fifecomptech.net`.

Run Linux commands in your SSH session on the new VM unless a step says to use your own computer. In `nano`, save with **Ctrl+O**, press **Enter** to confirm the filename, then exit with **Ctrl+X**.

> **Screenshot placeholder —** New Ubuntu VM overview with its public IPv4 address highlighted.

## 2. Install Docker Engine and Compose

### 2.1 Install from Docker’s official guide

Complete these sections of the [Docker Engine on Ubuntu tutorial](https://docs.docker.com/engine/install/ubuntu/) in order. The tutorial has the commands to copy into your VM’s SSH terminal.

1. **Prerequisites.** Check that your Ubuntu release is supported. Follow the conflicting-package instructions if your VM already has older or conflicting Docker packages.
2. **Install using the apt repository, step 1.** Add Docker’s signing key and APT repository. The key lets Ubuntu verify Docker’s packages. The repository tells APT where to find them.
3. **Install using the apt repository, step 2.** Install Docker Engine and the Compose plugin using the package list in the guide.
4. **Install using the apt repository, step 3.** Run Docker’s `hello-world` test with `sudo`. This downloads and runs a tiny container so you can confirm the installation works.

### 2.2 Set up your Docker group membership

Complete the **Manage Docker as a non-root user** section of [Docker’s Linux post-installation tutorial](https://docs.docker.com/engine/install/linux-postinstall/). The `docker` group gives root-level control of this VM, so add only your own trusted account.

1. Use the guide’s group creation command if the group does not already exist. Docker’s Ubuntu packages typically create it during installation.
2. Use the guide’s `usermod` command to append your account to the `docker` group. `$USER` expands to the username you logged in with.
3. Run `exit` to close the SSH session, then reconnect as you did in Tutorial 1. The new login picks up the group membership.
4. Run `docker run --rm hello-world` to check that Docker now works without `sudo`. The `--rm` flag removes the test container after it exits.
5. Run `docker compose version` to print the installed Compose plugin version.

> **Screenshot placeholder —** Successful Docker test without `sudo` and the Compose version.

## 3. Define Nextcloud with Docker Compose

An **image** packages software and its dependencies. A **container** is a running instance of an image. **Docker Compose** reads a YAML file that describes several containers, connects them on a private network and starts them together. A **volume** keeps data between container replacements.

This project uses the official Nextcloud image for the web app and MariaDB for the database. Compose’s private network lets Nextcloud reach the database by the service name `db`. Nginx on the VM will handle HTTPS and forward requests to Nextcloud through the VM’s loopback interface. [Official Nextcloud image](https://hub.docker.com/_/nextcloud/), [Nextcloud 33 requirements](https://docs.nextcloud.com/server/33/admin_manual/installation/system_requirements.html)

### 3.1 Create and open the project folder

```bash
mkdir -p ~/nextcloud
cd ~/nextcloud
pwd
ls -la
```

- `mkdir -p ~/nextcloud` creates the project folder in your home directory. The `-p` flag lets the command succeed if the folder already exists.
- `cd ~/nextcloud` moves your terminal into that folder.
- `pwd` prints the full path so you can check that it ends in `/nextcloud`.
- `ls -la` lists the folder contents, including files whose names start with a dot. A fresh folder shows only `.` and `..`.

### 3.2 Set your domain and database passwords in .env

Run `openssl rand -hex 24` twice to generate two different passwords. Each run prints 24 random bytes as 48 hexadecimal characters. Keep both results private. If `openssl` is missing, run `sudo apt install openssl` to install the command and try again.

```bash
touch .env
chmod 600 .env
nano .env
```

`touch` creates the file if it is missing. `chmod 600` gives your account read and write access to it. `nano` opens the file for editing.

Paste these three lines into `.env`, then replace all three placeholders before saving.

```dotenv
NEXTCLOUD_DOMAIN=YOUR DOMAIN HERE
DB_PASSWORD=YOUR FIRST GENERATED PASSWORD HERE
DB_ROOT_PASSWORD=YOUR SECOND GENERATED PASSWORD HERE
```

**Replace `YOUR DOMAIN HERE` with your full assigned hostname**, for example `NEXTCLOUD_DOMAIN=alex.fifecomptech.net`. Enter the hostname without `https://`, spaces or a trailing slash. Nextcloud uses `DB_PASSWORD` to connect to MariaDB. MariaDB uses `DB_ROOT_PASSWORD` for its own administrator account.

Run `ls -la` again to confirm that `.env` exists. Keep its passwords private and retain the file for future restarts.

> **Screenshot placeholder —** Project folder listing showing `.env`. Keep the file contents out of the screenshot.

### 3.3 Create the Compose file

Run `nano compose.yaml` to open the project’s Compose file. Paste the YAML below, keep the indentation, and save it.

```yaml
services:
  db:
    image: mariadb:11.4
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED
    environment:
      MARIADB_DATABASE: nextcloud
      MARIADB_USER: nextcloud
      MARIADB_PASSWORD: ${DB_PASSWORD:?Set DB_PASSWORD in .env}
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:?Set DB_ROOT_PASSWORD in .env}
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 30s

  app:
    image: nextcloud:33-apache
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "127.0.0.1:8080:80"
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: ${DB_PASSWORD:?Set DB_PASSWORD in .env}
      NEXTCLOUD_TRUSTED_DOMAINS: ${NEXTCLOUD_DOMAIN:?Set NEXTCLOUD_DOMAIN in .env}
      OVERWRITEPROTOCOL: https
      OVERWRITECLIURL: https://${NEXTCLOUD_DOMAIN:?Set NEXTCLOUD_DOMAIN in .env}
      PHP_UPLOAD_LIMIT: 100M
    volumes:
      - nextcloud_data:/var/www/html

volumes:
  db_data:
  nextcloud_data:
```

### 3.4 Check the file and download the images

```bash
ls -la
docker compose config --quiet
docker compose pull
```

- `ls -la` confirms that `.env` and `compose.yaml` are both in the current folder.
- `docker compose config --quiet` validates the Compose file and checks that required variables are present without printing the resolved configuration. Success produces no output.
- `docker compose pull` downloads the Nextcloud and MariaDB images. You’ll start the containers after enabling HTTPS.

## 4. Set up HTTPS with Nginx and Certbot

### 4.1 Check DNS and inbound access

In the Azure portal, open the new VM’s **Networking → Network settings** and check its effective inbound security rules. Allow public TCP **80** for certificate validation. During initial setup, allow TCP **443** from your computer’s public IPv4 address. Keep the SSH rule from Tutorial 1. If Ubuntu’s UFW firewall is already active, allow TCP 80 and 443 there too.

On your own computer, run the command below with your assigned hostname. `nslookup` asks DNS which IP address belongs to the name. Continue once it returns your new VM’s public IPv4 address.

```bash
nslookup subdomain.fifecomptech.net
```

> **Screenshot placeholder —** Azure inbound rules and DNS lookup for the assigned hostname.

### 4.2 Install the reverse proxy and certificate tools

```bash
sudo apt update
sudo apt install nginx
sudo apt install certbot python3-certbot-nginx
```

- `sudo` runs these system administration commands with administrator privileges.
- `apt update` refreshes Ubuntu’s package list.
- `apt install nginx` installs the reverse proxy that will accept web requests and forward them to Nextcloud.
- `apt install certbot python3-certbot-nginx` installs Certbot and the plugin that lets it configure HTTPS in Nginx.

### 4.3 Connect Nginx to Nextcloud

Run `sudo nano /etc/nginx/sites-available/nextcloud` to create the site configuration as an administrator. Paste this block and replace `subdomain.fifecomptech.net` on the `server_name` line with your assigned hostname. The proxy sends incoming requests to port 8080 on the VM, where the Nextcloud container will listen.

```nginx
server {
    listen 80;
    server_name subdomain.fifecomptech.net;
    client_max_body_size 100M;

    location = /.well-known/carddav {
        return 301 https://$host/remote.php/dav/;
    }

    location = /.well-known/caldav {
        return 301 https://$host/remote.php/dav/;
    }

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site and check its configuration. Use `ls -l` to show the enabled sites and where their links point. Run the `ln -s` command only if the `nextcloud` link is missing. On this fresh lab VM, run `unlink` only if the listing shows Ubuntu’s `default` symlink pointing to `/etc/nginx/sites-available/default`. This disables the welcome site while keeping its configuration file.

```bash
ls -l /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/nextcloud
sudo unlink /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```


- `ln -s` creates a link in `sites-enabled` so Nginx loads the site.
- `nginx -t` checks the configuration. The `&&` runs the reload only if the check succeeds.
- `systemctl reload nginx` applies the configuration to the running Nginx service.

Before requesting the certificate, run the following on the VM with your assigned hostname.

```bash
curl -I -H 'Host: subdomain.fifecomptech.net' http://127.0.0.1/
```

`-I` shows the response headers and `-H` sets the hostname Nginx uses to select the site. This checks the local Nginx setup without relying on public DNS. Expect **502 Bad Gateway** before the Nextcloud container starts. If you already set up HTTPS, expect a redirect to your HTTPS address. A **200 OK** showing the welcome page means you should check the enabled site and the `server_name` before continuing.

### 4.4 Issue the certificate on the new VM

Replace the hostname in both commands. Enter your email and accept the terms if Certbot prompts you. [Certbot user guide](https://eff-certbot.readthedocs.io/en/stable/using.html)

```bash
sudo certbot --nginx --redirect -d subdomain.fifecomptech.net
sudo certbot renew --dry-run --cert-name subdomain.fifecomptech.net
```

- `certbot --nginx --redirect -d` requests a Let’s Encrypt certificate for the domain after `-d`, configures Nginx to use it, and redirects HTTP requests to HTTPS.
- `certbot renew --dry-run --cert-name` simulates renewal of the named certificate so you can check that future renewal is set up.

Run `curl -I http://subdomain.fifecomptech.net` with your hostname. The `-I` flag displays the HTTP headers. The `Location` header should point to the HTTPS address. An HTTPS response of **502** at this stage means Nginx is waiting for the Nextcloud container.

> **Screenshot placeholder —** Successful certificate renewal test and HTTP-to-HTTPS redirect.

## 5. Start Nextcloud and create your admin account

```bash
cd ~/nextcloud
docker compose up -d
docker compose ps
```

- `cd ~/nextcloud` puts you in the folder that contains `compose.yaml`.
- `docker compose up -d` creates and starts the services. The `-d` flag leaves them running in the background.
- `docker compose ps` shows which services are running and the database health status.

Wait for the database to show **healthy**. The first app start may take a few minutes. Open your assigned `https://subdomain.fifecomptech.net` address in your browser to complete setup. Port 8080 is the internal HTTP backend for Nginx. Starting setup there, even through an SSH tunnel, can leave the browser unable to follow Nextcloud’s redirect to the assigned HTTPS address. Keep the HTTPS rule restricted to your computer until you have created the admin account.

1. Choose a Nextcloud administrator username and a strong, unique password.
2. Leave the data directory at its default. The database connection comes from Compose. If the form asks for database details, choose **MySQL/MariaDB**, set the database user and name to `nextcloud`, use the value of `DB_PASSWORD` from your private `.env`, and set the database host to `db`.
3. Click **Install**. Skip any optional recommended apps offered during setup for this storage lab.
4. Open **Files**. To create a regular account for everyday use, open **Users** from the profile menu.
5. If the lab requires access from other networks, update the Azure HTTPS rule to allow those connections now that the admin account exists.

> **Screenshot placeholder —** Nextcloud setup page with all credentials hidden.

## 6. Prove that your cloud storage works

1. In **Files**, create a folder named **Docker lab**.
2. Upload a small test file. Refresh the page, download it and open the downloaded copy.
3. Recreate the containers on the VM and check that the file survives.

   ```bash
   cd ~/nextcloud
   docker compose down
   docker compose up -d
   ```

   The first command opens the project folder. `docker compose down` stops and removes the containers while keeping their named volumes. `docker compose up -d` recreates the containers in the background using those same volumes. When Nextcloud is back, reopen your folder and check that your file is there. The `-v` flag on `down` would delete the volumes and their contents.

> **Screenshot placeholder —** The test file in Nextcloud Files after restarting the project.

### If you see “Welcome to nginx!”

1. Open the exact assigned HTTPS address in your browser, for example `https://alex.fifecomptech.net`. Run the DNS check in step 4.1 from your computer and confirm the result matches the new VM’s public IPv4 address.
2. On the new VM, repeat the checks in step 4.3. If necessary, run `sudo nginx -T` to print the active configuration. Check that it includes your assigned hostname and `proxy_pass http://127.0.0.1:8080;`. If you completed step 4.4, check that the server block listening on **443** includes the hostname and forwards requests to Nextcloud too.
3. After starting Nextcloud, test the HTTPS proxy from the VM. Replace the hostname in **both** places.

   ```bash
   curl --resolve subdomain.fifecomptech.net:443:127.0.0.1 https://subdomain.fifecomptech.net/status.php
   ```

   `--resolve` connects to local Nginx while keeping your domain for HTTPS certificate validation and site selection. Nextcloud’s `/status.php` should return JSON containing an `installed` field. If it works locally but the browser still shows the welcome page, recheck public DNS for an old address, including any IPv6 record. If the local test returns **502**, check the container status and logs below.

### If setup redirects to “Page cannot be reached”

The `/index.php/core/apps/recommended` page is part of Nextcloud’s first-run setup. If the browser cannot reach it, first check whether it can reach the same HTTPS hostname at all. Replace the example hostname in the commands below.

1. On your own computer, run:

   ```bash
   curl -v --connect-timeout 10 https://subdomain.fifecomptech.net/status.php
   ```

   `-v` shows where the connection stops. `--connect-timeout 10` limits the connection attempt to ten seconds. A DNS error means you should check the domain as in step 4.1. A timeout with the correct IP means you should check the Azure TCP **443** rule, including whether its source matches your current public IP, and the VM firewall.
2. If a firewall blocks the browser test, open the VM in the Azure portal and select **Operations → Run command → RunShellScript**. Paste the local HTTPS test below, replace the hostname in both places, and select **Run**. If you already have an SSH session to the VM, you can run the same command there.

   ```bash
   curl --connect-timeout 5 --resolve subdomain.fifecomptech.net:443:127.0.0.1 https://subdomain.fifecomptech.net/status.php
   ```

   This makes the connection inside the VM while checking the hostname and certificate. It distinguishes a working local service from a public network access problem. If it cannot connect to local port 443, check whether Nginx has a listener:

   ```bash
   sudo ss -ltnp 'sport = :443'
   ```

   `ss` shows listening TCP sockets and their owning processes. The filter selects port 443. If Nginx is missing from the output, check its configuration in step 4.3 and complete the HTTPS setup in step 4.4. If the local HTTPS test returns Nextcloud JSON but the public test fails, check DNS and inbound access before changing Docker.
3. If the public test returns JSON with `"installed":true`, open the main HTTPS page in a fresh browser tab and sign in with the account you already created. Skip the optional recommended apps if prompted.

### If something else goes wrong

| Symptom | What to check |
| --- | --- |
| DNS resolves incorrectly | Ask your tutor to check the DNS record against the public IP on the active VM. Check for a stale IPv6 record if IPv6 was previously configured. |
| Certbot cannot validate the name | Check public DNS and inbound TCP 80. Run `sudo nginx -t` to check for Nginx configuration errors. |
| HTTPS still gives 502 | In the project folder, use `docker compose ps -a` to see all containers, including stopped ones. Use `docker compose logs --tail=80 app db` to show the last 80 log lines from each service. |
| The database is unhealthy | Read the DB logs. If the database already initialized, changing `.env` alone does not change its stored password. |
| The VM becomes slow or a container exits unexpectedly | Use `free -h` to see available memory, `df -h` to see available disk space, and `docker stats --no-stream` for a one-time view of container resource use. The `-h` flag displays readable units. |

To try browser-based editing after completing the storage lab, see the [Community Document Server extension](community-document-server.md).
