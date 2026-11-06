OpenUDS
=======

OpenUDS (Universal Desktop Services) is a multiplatform connection broker for:
- VDI: Windows and Linux virtual desktops administration and deployment
- App virtualization
- Desktop services consolidation
- ...

This is an Open Source project, initiated by Spanish Company ​Virtual Cable and released Open Source with the help of several Spanish Universities.

Please feel free to contribute to this project.

Notes
=====
* Master version is always under heavy development and it is not recommended for use, it will probably have unfixed bugs.  Please use the latest stable branch (`v4.0` right now).
* From `v4.0` onwards (current master), OpenUDS has been splitted in several repositories and contains submodules. Remember to use "git clone --resursive ..." to fetch it ;-).
* `v4.0` version needs Python 3.11 (may work fine on newer versions). It uses new features only available on 3.10 or later, and is tested against 3.11. It will probably work on 3.10 too.

Installation
============
There is an `install.sh` file which will run through this automatically, but I also documented each command run and what the command does for those interested or concerned.

Steps may differ per repo, but I'm using Rocky Linux 10.2 for refference. If possible, use a fresh install; I did mine as a minimal install with headless management tools.

### Prerequisites and environment setup
* Install required tools
  * `sudo dnf install -y epel-release wget mariadb mariadb-server mariadb-devel python3.12-devel libmemcached-awesome-devel libxml2-devel nginx && sudo dnf groupinstall -y "Development Tools"`
* Create a new user for permission management
  * `sudo useradd --system --home-dir /opt/openuds --shell /sbin/nologin openuds`
  * `sudo usermod -aG openuds nginx`
* Clone this repo into your user's home directory
  * `cd ~ && git clone https://github.com/Westley-R/openuds.git`
* Pull the submodules. As of writing, no URL is found for freckle-references; this is a known bug, and is probably fine.
  * `git submodule update --init --recursive`
* Move openuds to /opt folder
  * `sudo mv openuds /opt`
* Make a python 3.12 environment and activate it
  * `cd /opt/openuds && sudo -u openuds python3 -m venv .env`
* Update pip and install python requirements for openuds-server 
  * `.env/bin/python -m pip install --upgrade pip`
  * `source .env/bin/activate`
  * `pip install -r requirements.txt`

### MariaDB Setup
* Enable mariadb 
  * `sudo systemctl enable --now mariadb`
* Secure MariaDB (can be skipped)
  * `mysql_secure_installation`
  * `Enter current password for root (enter for none): <hit enter>`
* Create a new mariadb user, dbuds database, and grant the new user privileges to it.
  * Generate a random password for your mariadb user; this step can be skipped if you'd rather use a custom password.
    * `openssl rand -base64 32`
  * `sudo mariadb -u root -p`
  * `CREATE DATABASE dbuds;`
  * `CREATE USER 'dbuds'@'localhost' IDENTIFIED BY '<INSERT PASSWORD HERE>';`
  * `GRANT ALL PRIVILEGES ON dbuds.* TO 'dbuds'@'localhost';`
  * `FLUSH PRIVILEGES;`
  * `exit`
* Update the openuds server settings to account for your mariadb server
  * `cd /opt/openuds/server/src/server && cp settings.py.sample settings.py`
  * Updating `settings.py` will vary from user to user; I just changed RSA_KEY based on the instructions in the comment above it, and PASSWORD to my randomized password used to create the mariadb user.
* Run the openuds migrate command to create missing database tables 
  * `cd /opt/openuds/server/src && mkdir log && python manage.py migrate`

### nginx setup
* Generate a self-signed https certificate; you can skip/adjust some of these steps if you have an alternate way of getting a certificate.
  * `sudo openssl req -x509 -nodes -newkey rsa:4096 -days 825 -keyout /etc/pki/tls/private/uds.key -out /etc/pki/tls/certs/uds.crt -subj "/CN=uds.local"`
* Copy the default config and edit to adjust to your setup. Default should be fine for most things, just be sure to change `uds.local` to your prefered `hostname.lan`/`hostname.workgroup`
  * `sudo mv /opt/openuds/openuds-nginx.default.conf /etc/nginx/conf.d/openuds-nginx.conf`
* Test nginx config and start the background process
  * `sudo nginx -t && sudo systemctl enable --now nginx`
* Configure your firewall rules to allow connection to the nginx server; the default should work for most setups.
  * `sudo firewall-cmd --permanent --add-service={http,https}`
  * `sudo firewall-cmd --reload`
* Setup static https files
  * `sudo -u openuds /opt/openuds/.env/bin/python manage.py collectstatic`
  * `sudo semanage fcontext -a -t httpd_sys_content_t "/opt/openuds/server/src/static(/.*)?"`
  * `sudo restorecon -Rv /opt/openuds/server/src/static`

### Final setup
* `sudo mv /opt/openuds/*.service /etc/systemd/system/`
* `sudo systemctl daemon-reload && sudo systemctl enable --now openuds-web`

*Warning:* DeepWiki is AI generated based on an older version of the primary repo; it's good for general information, but version specific instructions are all wrong.

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/VirtualCable/openuds)
