## LetsEncrypt

### Certbot Installation

- [reference](https://certbot.eff.org/lets-encrypt/ubuntuxenial-other)

This procedure describes the commands for Ubuntu 16.04.  
If you use another OS, use the reference link, choose *software: None of the above* and your OS and follow the installation instructions.

```bash
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot
```

### Generate certificate using DNS validation

- [Reference](https://certbot.eff.org/docs/using.html#manual)

Make sure your DNS supports letsencrypt CAA by verifying that it has this field in its config file:

```json
  "certificateAuthorityAuthorization": {
    "issuer": "letsencrypt.org"
  },
```

If you are not familiar with this process, it is recommended to do a dry-run as the LetsEncrypt API has a call limit, which may block you in case of multiple failed attempts. For this, append `--dry-run` to the command below. Once it works, simply repeat it without `--dry-run`.

Launch the process using: `certbot certonly --manual --preferred-challenges dns`

When prompted for the domain, enter `*.DOMAIN` and accept to share the IP address by pressing `ENTER`.

Now, the CLI will ask you to set a certain key to the TXT Record `_acme-challenge`. Enter it in the DNS config by adding the following field under `dns:staticDataInDomain`:

```json
{
  // ...
  "dns": {
    // ...
    "staticDataInDomain": {
      "_acme-challenge": {
        "description": "REPLACE_WITH_ACME_CHALLENGE"
      },
      // ...
  }
}
```

and reboot the DNS service:

```bash
docker stop pryv_dns
./run-pryv
```

Verify that the key is set by running `dig TXT _acme-challenge.DOMAIN`, it may require multiple runs to reach the right result (`dig @dns1.DOMAIN TXT _acme-challenge.DOMAIN` is maybe better here). Once you get the right key, go back to the CLI and press ENTER.

You should now have a certificate in `/etc/letsencrypt/live/DOMAIN/`.

#### Reorganize SSL certificate files

Rename the files to match the NGINX settings:

```
mv fullchain.pem DOMAIN-bundle.crt
mv privkey.pem DOMAIN-key.pem
```

You might have to copy them as `live/` holds symbolic links

Then copy them into: `${PRYV_CONF_ROOT}/pryv.io/single-node/nginx/conf/secret/`.

and reboot the NGINX service:

```bash
docker stop pryv_reverse_proxy
./run-pryv
```