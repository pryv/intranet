# Benchmarking core docker with Chrome inspector

- SSH to the machine hosting the docker container you want to monitor. 
- Get CONTAINER_ID: `core$ sudo docker ps`
- Fetch INTERNAL_CONTAINER_IP_ON_TARGET (look for Networks/backend/IPAddress): `core$ sudo docker inspect CONTAINER_ID`
- Create an SSH tunnel from localhost to the corresponding port on the container: `(your machine)$ ssh -L 127.0.0.1:9229:INTERNAL_CONTAINER_IP_ON_TARGET:9229 USER@core1.bdb.pryv.tech`

- If a process was started without --inspect, signal it with SIGUSR1 to activate the debugger and print the connection URL. (https://nodejs.org/en/docs/inspector/)
  - `core$ sudo htop`
  - Find the _node process_ (usr/bin/node src/server --config/app/conf/core.json)
  - Select it with spacebar
  - F9 to send SIGUSR1 (signal #10, activate node debug mode)
  - Verify in docker logs: `core$ sudo docker logs -f CONTAINER_ID`

- Additional proxy with socat (Why: SIGUSR1 will bind the inspector port to 127.0.0.1:9229, but we want to reach it on INTERNAL_CONTAINER_IP_ON_TARGET. So we have to proxy between these).
  - `core$ sudo docker exec -ti DOCKER_CONTAINER /bin/bash`
  - If needed: `apt-get update`, `apt-get install socat`
  - `root@CONTAINER_ID$ socat -d -d -lmlocal2 \` `TCP4-LISTEN:9229,bind=INTERNAL_CONTAINER_IP_ON_TARGET,reuseaddr,fork,su=nobody \` `TCP4:127.0.0.1:9229`
  
## In chrome
- Open: _chrome://inspect_
- Click **Configure...** and check that _localhost:9229_ record is present
- Check **Discover network targets**
- Under **Remote Target** you should see your node target (src/server), click **Inspect**
- Click **Start** to start an inspection
- Click **Stop** to stop the inspection
- Analyse the generated .cpuprofile file