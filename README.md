# freenas-iocage-nextcloud
Script to create an iocage jail on FreeNAS for the latest Nextcloud 16 release, including Caddy 1.0, MariaDB 10.3/PostgreSQL 10, and Let's Encrypt

This script will create an iocage jail on FreeNAS 11.1 or 11.2 with the latest release of Nextcloud 16, along with its dependencies.  It will obtain a trusted certificate from Let's Encrypt for the system, install it, and configure it to renew automatically.  It will create the Nextcloud database and generate a strong root password and user password for the database system.  It will configure the jail to store the database and Nextcloud user data outside the jail, so it will not be lost in the event you need to rebuild the jail.

## Status
This script appears to work well on all FreeNAS 11.2 releases.  FreeNAS 11.2-U2.1 (and perhaps -U2) have a bug that results in jail mountpoints being lost on system restart.  The most common indication of this is that the Nextcloud page doesn't appear when you browse to your installation.  If you experience this, first run `iocage fstab -l nextcloud`.  If you don't see four entries, you've been bitten.  To recover, enter the jail's shell.  Run `service mysql-server stop` followed by `rm -rf /var/db/mysql/*`.  Then exit the shell and stop the jail.

Next, re-add the mountpoints, either through the FreeNAS GUI or at the shell, whichever you prefer.  $DB_PATH (which by default is $POOL_PATH/db) needs to be mounted at /var/db/mysql/.  $FILES_PATH (which by default is $POOL_PATH/files) needs to be mounted at /media/files.  Then restart the jail and you should be good to go.

## Usage

### Prerequisites
Although not required, it's recommended to create two datasets on your main storage pool: one named `files`, which will store the Nextcloud user data; and one called `db`, which will store the SQL database.  For optimal performance, set the record size of the `db` dataset to 16 KB (under Advanced Settings).  It's also recommended to cache only metadata on the `db` dataset; you can do this by running `zfs set primarycache=metadata poolname/db`.

### Installation
Download the repository to a convenient directory on your FreeNAS system by running `git clone https://github.com/danb35/freenas-iocage-nextcloud`.  Then change into the new directory and create a file called `nextcloud-config`.  It should look like this:
```
JAIL_IP="192.168.1.199"
DEFAULT_GW_IP="192.168.1.1"
POOL_PATH="/mnt/tank"
TIME_ZONE="America/New_York" # See http://php.net/manual/en/timezones.php
HOST_NAME="YOUR_FQDN"
STANDALONE_CERT=0
DNS_CERT=0
CERT_EMAIL="me@example.com"
```
Many of the options are self-explanatory, and all should be adjusted to suit your needs, but only a few are mandatory.  The mandatory options are:

* JAIL_IP is the IP address for your jail
* DEFAULT_GW_IP is the address for your default gateway
* POOL_PATH is the path for your data pool.
* TIME_ZONE is the time zone of your location, in PHP notation--see the [PHP manual](http://php.net/manual/en/timezones.php) for a list of all valid time zones.
* HOST_NAME is the fully-qualified domain name you want to assign to your installation.  You must own (or at least control) this domain, because Let's Encrypt will test that control.
* DNS_CERT and STANDALONE_CERT determine which method will be used to validate domain control for Let's Encrypt.  One **and only one** of these must be set to 1.
* DNS_PLUGIN: If DNS_CERT is set, DNS_PLUGIN must contain the name of the DNS validation plugin you'll use with Caddy to validate domain control.  See the [Caddy documentation](https://caddyserver.com/docs) under the heading of "DNS Providers" for the available plugins, but omit the leading "tls.dns.".  For example, to use Cloudflare, set `DNS_PLUGIN="cloudflare"`.
* DNS_ENV: If DNS_CERT is set, DNS_ENV must contain the authentication credentials for your DNS provider.  See the [Caddy documentation](https://caddyserver.com/docs) under the heading of "DNS Providers" for further details.  For Cloudflare, you'd set `DNS_ENV="CLOUDFLARE_EMAIL=foo@bar.baz CLOUDFLARE_API_KEY=blah"`.
* CERT_EMAIL is the email address Let's Encrypt will use to notify you of certificate expiration
 
In addition, there are some other options which have sensible defaults, but can be adjusted if needed.  These are:

* JAIL_NAME: The name of the jail, defaults to "nextcloud"
* DB_PATH, FILES_PATH, and PORTS_PATH: These are the paths to your database files, your data files, and the FreeBSD Ports collection.  They default to $POOL_PATH/db, $POOL_PATH/files, and $POOL_PATH/portsnap, respectively.
* DATABASE: Which database management system to use.  Default is "mariadb", but can be set to "pgsql" if you prefer to use PostgreSQL.
* INTERFACE: The network interface to use for the jail.  Defaults to `vnet0`.
* VNET: Whether to use the iocage virtual network stack.  Defaults to `on`.

If you're going to open ports 80 and 443 from the outside world to your jail, do so before running the script, and set STANDALONE_CERT to 1.  If not, but you use a DNS provider that's supported by Caddy, set DNS_CERT to 1.  If neither of these is true, you won't be able to use this script without modification.

It's also helpful if HOST_NAME resolves to your jail from **inside** your network.  You'll probably need to configure this on your router.  If it doesn't, you'll still be able to reach your Nextcloud installation via the jail's IP address, but you'll get certificate errors that way.

### Execution
Once you've downloaded the script and prepared the configuration file, run this script (`./nextcloud-jail.sh`).  The script will run for several minutes.  When it finishes, your jail will be created, Nextcloud will be installed and configured, and you'll be shown the randomly-generated password for the default user ("admin").  You can then log in and create users, add data, and generally do whatever else you like.

This configuration generated by this script will obtain certs from a non-trusted certificate authority by default.  This is to prevent you from exhausting the [Let's Encrypt rate limits](https://letsencrypt.org/docs/rate-limits/) while you're testing things out.  Once you're sure things are working, you'll want to get a trusted cert instead.  To do this, enter the jail by running `iocage console nextcloud`.  Then edit the Caddyfile by running `nano /usr/local/www/Caddyfile`.  Near the top, you'll see a block that says (if you used DNS validation):
```
	tls {
		ca https://acme-staging-v02.api.letsencrypt.org/directory
		DNS-PLACEHOLDER
	}
```
Or, if you didn't use DNS validation:
```
	tls {
		ca https://acme-staging-v02.api.letsencrypt.org/directory
	}
```
If you used DNS validation, remove the line that says `ca https://acme-staging-v02.api.letsencrypt.org/directory`.  If you didn't use DNS validation, remove the entire block.  Then restart Caddy using `service caddy restart`.

### To Do
I'd appreciate any suggestions (or, better yet, pull requests) to improve the various config files I'm using.  Most of them are adapted from the default configuration files that ship with the software in question, and have only been lightly edited to work in this application.  But if there are changes to settings or organization that could improve performance or reliability, I'd like to hear about them.

### HTTP Strict Transport Security (HSTS)
Nextcloud will warn you that HSTS is not enabled.  This is intentional--if there's a problem with your TLS configuration, your cert expires, etc., HSTS can prevent you from accessing your site.  Once you're sure everything is working properly with TLS and your certificate (including renewal if you're using a Let's Encrypt cert), you should enable it by uncommenting line 43 in `/usr/local/etc/apache24/Includes/$HOST_NAME.conf`.
