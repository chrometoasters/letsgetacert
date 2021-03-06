# Let's Get a Cert
A [Certbot](https://certbot.eff.org/) wrapper to process certificate configuration files and to generate certificates if needed. Uses pre-generated private key to generate CSR, the key is not overwritten so you can use HTTP Public Key Pinning (HPKP) with your website.

# Prerequisites
1. [Certbot](https://certbot.eff.org/) installed
2. A webserver with HTTPS support enabled, make sure `.well-known` directory in the document root is accessible and the URLs are not rewritten

# Installation
1. Clone this repository somewhere
2. Copy `letsgetacert.cnf.template` to `letsgetacert.cnf`
3.  Edit `letsgetacert.cnf`, see configuration options below
4. Create certificate configuration files for your certificates and domains
5. Run `letsgetacert --verbose --no-cert` to see if the certificate configuration files are processed correctly
6. Run `letsgetacert --verbose` to get your certificates if needed
7. If everything is ok place `letsgetacert` to your crontab so that it get's executed daily; no need to run with superuser privileges, `sudo` is used when needed

## Usage
```
letsgetacert [-c|--config CONFIGFILE] [-e|--list-expires] [-f|--force COMMONNAME] [-n|--no-cert] [-v|--verbose]
```

```
-c, --c CONFIGFILE
```
Read `CONFIGFILE` instead of `letsgetacert.cnf` in the `letsgetacert` directory

```
-e, --list-expires
```
Only list expire dates; `--force` and `--no-cert` do nothing when used together with `--list-expires`

```
-f, --force COMMONNAME
```
Get certificate for `COMMONNAME` even if it's not expired yet, for example when you want to add a new SAN (Subject Alternative Name)

```
-n, --no-cert
```
Don't generate a CSR, don't get a certificate, like a *dry run* for testing

```
-v, --verbose
```
Be verbose and report what's going on

## Configuration file options
### Example file
```
EXPIRY_THRESHOLD=5
CONFDIR=/home/ubuntu/.letsgetacert
CERTBOT=/opt/certbot/certbot-auto
CERTBOT_EXTRA_OPTS="--test-cert --quiet"
SUBJECT_EMAIL=bot@example.com
function hook {
    sudo service nginx reload
}
```

### Options with example values
```
EXPIRY_THRESHOLD=5
```
Renew certs this many days before the expiration date

```
CONFDIR=/home/ubuntu/.letsgetacert
```
Where to look for the certificate config files

```
CERTBOT=/opt/certbot/certbot-auto
```
How to execute Certbot

```
CERTBOT_EXTRA_OPTS="--test-cert --quiet"
```
Certbot extra options, `--test-cert` is for generating invalid, testing certificates

```
SUBJECT_EMAIL=bot@example.com
```
Email to be used in certificate subject field

```
function hook {
    sudo service nginx reload
}
```
Call this function when at least one cert was generated; use it to reload your web server configuration; you can also use these variables in the hook function:

- `$NEW_CERTS_CN`: array of common names from generated certificates
- `$NEW_CERTS_EXPIRY`: array of expiration dates of the newly generated certificates, in seconds since the epoch

## Example certificate configuration file
Name the file after the `CN` field, use `.cnf` extension when saving and place the file in the `$CONFDIR` directory, in this case the filename should be `example.com.cnf`
```
CN=example.com
PRIVKEY=/etc/nginx/certs/$CN.privkey.pem
CERT_DIR=/etc/nginx/certs
SUBJECT="/C=CZ/ST=Prague/L=Prague/O=example.com/emailAddress=$SUBJECT_EMAIL/CN=$CN"
WEBROOT_DIR=/srv/www/$CN/site/public
DOMAINS="www=example.com,www.example.com;foo=foo.example.com;bar=bar.example.com"
```

### Options
```
CN=example.com
```
Certificate common name

```
PRIVKEY=/etc/nginx/certs/$CN.privkey.pem
```
Path to private key, must exist before generating certs, will not be overwritten

```
CERT_DIR=/etc/nginx/certs
```
Where to put generated certs, must contain `archive` subdirectory; files are put into the `archive` directory, `$CERT_DIR` will contain symbolic links

```
SUBJECT="/C=CZ/ST=Prague/L=Prague/O=example.com/emailAddress=$SUBJECT_EMAIL/CN=$CN"
```
Certificate subject; you can use `$SUBJECT_EMAIL` and `$CN` variables

```
WEBROOT_DIR=/srv/www/$CN/site/public
```
Path to where the web root directories are placed

```
DOMAINS="www=example.com,www.example.com;foo=foo.example.com;bar=bar.example.com"
```
Configuration of domains for the certificate, these will be placed in the Subject Alternative Name (SAN) field; the format is `DIR=DOMAINS`, Certbot will look for verification files in `$WEBROOT_DIR/DIR/.well-known` directory for `DOMAINS`; separate multiple domains with comma (`,`), multiple `DIR=DOMAINS` with semicolon (`;`)

## Seamless transition
You can use your existing certificates and keys with `letsgetacert`, just create a symbolic link in the `CERT_DIR`. The name should follow this pattern: `$CN.fullchain.pem`. Then the file will be picked up by `letsgetacert` automatically and if the certificate is going to expire soon it will be renewed using Certbot.
