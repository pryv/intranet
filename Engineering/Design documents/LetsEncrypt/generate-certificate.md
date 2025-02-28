## LetsEncrypt

### Certbot Installation

- [reference](https://certbot.eff.org/lets-encrypt/ubuntuxenial-other)

This procedure describes the commands for Ubuntu 16.04.  
If you use another OS, use the reference link, choose *software: None of the above* and your OS and follow the installation instructions.

```
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot
```

### Generate certificate using DNS validation

- [Reference](https://certbot.eff.org/docs/using.html#manual)

Make sure your DNS supports letsencrypt CAA by verifying that it has this field in its platform variables:

```yaml
  ADVANCED_API_SETTINGS:
    optional: true
    name: "Advanced API settings"
    settings:
      SSL_CAA_ISSUER: 
        value: letsencrypt.org
        description: "Certificate authority allowed to issue SSL certificates for this domain"
```

If you are not familiar with this process, it is recommended to do a dry-run as the LetsEncrypt API has a call limit, which may block you in case of multiple failed attempts. For this, append `--dry-run` to the command below. Once it works, simply repeat it without `--dry-run`.

Launch the process using: `certbot certonly --manual --preferred-challenges dns`

When prompted for the domain, enter `*.DOMAIN` and accept to share the IP address by pressing `ENTER`.

Now, the CLI will ask you to set a certain key to the TXT Record `_acme-challenge`. Enter it in the platform variables by adding the following field under `DNS_SETTINGS:settings:DNS_CUSTOM_ENTRIES`:

```yaml
  DNS_SETTINGS:
    name: "DNS settings"
    settings:
      DNS_CUSTOM_ENTRIES:
        description: "Additional DNS entries. See the DNS configuration document: https://api.pryv.com/customer-resources/#documents.
        Can be set to null if not used."
        value: 
          _acme-challenge:
            description: "KEY"
```

and reboot the follower and pryv services.

Verify that the key is set by running `dig TXT _acme-challenge.DOMAIN`, it may require multiple runs to reach the right result (`dig @dns1.DOMAIN TXT _acme-challenge.DOMAIN` is maybe better here). Once you get the right key, go back to the CLI and press ENTER.

You should now have a certificate in `/etc/letsencrypt/live/DOMAIN/`.

#### Reorganize SSL certificate files

Rename the files to match the NGINX settings:

```
mv fullchain.pem DOMAIN-bundle.crt
mv privkey.pem DOMAIN-key.pem
```

You might have to copy them as `live/` holds symbolic links

Then copy them into: `${PRYV_CONF_ROOT}/config-leader/data/${ROLE}/nginx/conf/secret/` with `${ROLE}` being:

- `singlenode`

OR

- `core`
- `reg-master`
- `reg-slave`
- `static`

and reboot the follower and pryv services.