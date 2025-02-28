## Flow

- login using credentials in `config-leader.json:credentials:filePath`
- fetch platform settings
- obtain DOMAIN
- start DNS challenge for DOMAIN
- receive KEY
- update platform settings: set KEY in platform.yml `DNS_CUSTOM_ENTRIES`
- tell leader to update DNS service
- verify that it is set (dig TXT _acme-challenge.DOMAIN)
- Continue process (CA checks for _acme-challenge.DOMAIN)
- Obtain certificates
- write them to `config-leader.json:templatesPath` + `/ROLE/nginx/conf/secret/`
- Tell all followers to fetch new config & reboot NGINX

## Requirements

1. config-leader valid token - credentials from `config-leader.json:credentials:filePath`
2. access to `config-leader/data/` to write cert

### First boot

1. setup platform.yml
2. run config-lead
3. run config-fol
4. run pryv
5. launch SSL cert generation

## Tasks

### Leader

1. update route `/admin/notify` to accept optional **services** parameter

### Follower

1. update route `/notify` to accept optional **services** parameter and reboot only the required services

### Other container or leader

Implement flow with Letsencrypt




