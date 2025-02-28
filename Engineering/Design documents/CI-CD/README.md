# Automate Pryv.io builds

if any phase fails, stop process, maybe notify users

1. run tests
2. update changelog
3. Push changes to the feature branch

**Automation starts** 
(For **Service-Core** and **Service-Register**)

5. **Auto**: With pre-commit hook Enterprise licenses are added to all files
6. Developer should commit them separately if there are any new ones
7. **Auto**: On pull request to master tests are run by github workflow on the branch
8. PR
9. **Auto**: On merge to master tests are run again on master with new merged code
10. On Release: Checkout to master, download all the changes, git describe to get latest tag.
Choose the newer tag, create and push it.
11. **Auto**: On new tag, github worflow will run tests and will build code and push new images
 to the conteiners registry

### Open Pryv.io
1. Create new tag and push it to the node-opener master branch
2. **Auto** Building, testing and publishing pipeline. P.s. now `yarn build` builds does the
 following:
    1. Strip down functionalities
    2. Overwrite some register code
    3. Overrides licenses in the dest folder so that released code has open licenses
    4. Push to Open Pryv.io repository

### Codebase
1. Added "unlicensed" in the package.json
2. Changed building logic - not to build .tar file, but to have .dockerignore file and build
 directly from the context

### Important notes
1. In the service-core private lib-reporting package is used and to have private dependencies
 inside docker image several solutions were implemented:
    
    a) for the pipelines - node_modules are installed before building the image
    
    b) for running yarn install in the image Docker.intermediate files were created and they
     create intermediate image with ssh authentication. This may be useful when building the
      image not in the github pipeline
 2. When setting up environment for development, third parties software are run in the
  background (redis, mongodb, influx, gnatsd). The same behaviour will appear in the dev
   environment.

### Tools selection 
Github workflows were chosen, because it works almost out of the box and is free (https://github.com/pricing 2,000 Actions minutes/month for free plan and 3,000 Actions minutes/month for
 team plan). Github and Jenkins had overlaping functionalities so to simplify process small
  build sccripts were moved to Github worflows. If later private servers would be needed (I
   guess it is only the case if commits are very often and 3000 minutes are not enough
  ), self-hosted functionalities could be used explanation is here ( https://help.github.com/en
  /actions/hosting-your
  -own-runners/about-self-hosted-runners )
  
### How to setup and use github worflow interfase
1. To find worflows go to the repository and click on actions in the top menu tab (for example
: https://github.com/pryv/service-core/actions )
2. To setup new tokens - first get the token for pryvbot user (email is ieva@pryv.com ). Now
 there are 2 tokens - one with reading access and another with reading and writing access. These
  tokens could be added to each repository as secrets - go to repository->Settings->Secrets (you
   need Admin access to see this selection) and create or update the secrets. SSH key is created
    also for Pryvbot so it is better not to give too push access for this user.

### CI/CD secrets

To connect to docker registry, usually these 3 variables are used:

1. REGISTRY_PRYV_PASSWORD=content of json file for service account
2. REGISTRY_PRYV_SERVER=eu.gcr.io
3. REGISTRY_PRYV_USERNAME=_json_key
