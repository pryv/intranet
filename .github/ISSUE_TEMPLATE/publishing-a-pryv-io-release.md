---
name: Publishing a pryv io release
about: Publishing a pryv io release
title: ''
labels: ''
assignees: ''

---

## Make release candidate (optional)

- [ ] Tag repo(s) with RC version to build image(s): `<major>.<minor>.<patch>-rc<number>`
- [ ] Validate RC on staging pryv.me (edit `pryv.yml` directly on the machine)
   - [ ] On pryv.me create a new user from [PRYV ACCESS TOKEN GENERATION](https://api.pryv.com/app-web-access/)
   - [ ] Open [Mariana2 account](https://mariana2.pryv.me/) and go to 2020 to see if all data are here (check browser console)
   - [ ] Open [Demo pages](https://demo.pryv.me/) check all tabs
   - [ ] Run jslib tests
   

## Publish

- [ ] Tag repo(s) with version to build image(s)

- **In [config leader](https://github.com/pryv/service-config-leader):**
   - [] Add the following files in `src/controller/migration/scriptsAndTemplates/{mode}/` (find inspiration from previous entries):
      - `${version}.js`
      - `${version}-template.yml` (make sure you bump `TEMPLATE_VERSION` and add possible new settings in there)
      - Notes: If there is no difference between cluster and single-node, write only the `.js` for cluster and put the following in single-node:
        ```javascript
        module.exports = require('../cluster/1.7.0');
        ```
        If you need to manipulate some object, use [utils.getObjectOrParseJSON()](https://github.com/pryv/service-config-leader/blob/master/src/controller/migration/scriptsAndTemplates/utils.js#L111) to parse it, for it may be JSON or not.
   - [] Add links to the files you added in the `migrations` object in `src/controller/migration/migrations.js`
   - [] Run tests
   - [] Update version in `package.json`
   - [] Update `CHANGELOG`
   - [] Commit with message `template {release version}`
   - [] Tag as `{bumped leader version}` with message `template {release version}` (triggers building of new Docker image)
   - [] Push

- **In [config template](https://github.com/pryv/config-template-pryv.io):**

   - [] If needed, update the platform templates by copying the new versions over from config leader
   - [] If needed, update service configuration(s) (e.g. new settings), referring to corresponding template settings if appropriate
   - [ ] Update Docker image versions in `{mode}/config-leader/config-leader.yml` for `config-leader` and any other updated service
   - [ ] Commit with message `upgrade leader[, {other updated services}] with {release version} upgrade and bump version`
   - [ ] Tag as `{release version}` (with message summarizing changes)
   - [ ] **TO REVIEW:** Validate on pryv.li: merge template into config-pryv.li & apply on machines
   - [ ] **TO REVIEW:** Install on pryv.me: merge template into config-pryv.me & apply on machines
   - [ ] Publish tarballs by running `./{mode}/scripts/publish.sh` for both cluster and single-node (the script uses `sudo` because it performs a `chown` of the config files to the user:group `9999:9999`, which runs our Pryv.io services inside the containers)
   - [ ] Commit with message `publish {release version}`
   - [ ] Push
