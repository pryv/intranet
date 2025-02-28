## Context

Titre: Mise à jour plateforme pryv.addmin.com

Bonjour,

Comme discuté avec [Stéphane Journot](mailto:stephane@addmin.com) (en CC), nous voudrions procéder à la mise à jour de la plateforme pryv.addmin.com (IP: 141.101.62.192) mardi 16.03.

Pour cela, nous voudrions au préalable tester la mise à jour sur une copie des données. Pouvez-vous pour cela exécuter les actions suivantes. Cela provoquera un down time de quelques secondes du service Pryv.io (le temps de les reboot), pour lequel il faudra se synchroniser avec Addmin. Si possible faire ça avant lundi midi.

1. Créer un dossier /var/pryv1.6 avec les droit pour nos users
2. Eteindre Pryv.io 
   1. `/var/pryv/stop-core`
   2.   
   3. `/var/pryv/stop-config-follower`
   4. `/var/pryv/stop-config-leader`
3. `mv /var/pryv en /var/pryv1.5 -- sudo switch_pryv.sh 1.6`
4. Lancer Pryv.io
   1. `/var/pryv/start-config-leader`
   2. `/var/pryv/start-config-follower`
   3. `/var/pryv/run-pryv`
5. De créer un fichier `switch_pryv.sh` executable en sudo avec le contenu suivant (pour qu’on puisse exécuter `sudo switch_pryv.sh`):  
    `rm -f /var/pryv; ln -s /var/pryv${1}  /var/pryv`
    
Suite à cela, nous voudrions exécuter la procédure de mise à jour sur une copie des données dans `/var/pryv1.6`

Si les containers core et mongodb démarrent comme il faut, nous appliquerons la mise à jour de la même manière le lendemain.

## Upgrade procedure 

1. prepare `/var/pryv2` for new version
   1. Copy needed files from `/var/pryv`
       1. List here 
2. Stop all services but mongodb
3. Mongodump
   1. Check that no `pryv-node` folder exists in `/var/pryv/pryv/mongodb/backup`
   2. Run
      `docker exec -ti pryvio_mongodb /app/bin/mongodb/bin/mongodump --drop -d pryv-node -o /app/backup`
4. Stop mongo
5. `rsync -av /var/pryv/pryv/core/data/ /var/pryv1.6/pryv/core/data/`
6. Rsync `/var/pryv` to `/var/pryv1`
7. Move `/var/pryv2` to `/var/pryv`
8. Migrate REDIS 

`sudo /usr/local/sbin/switch_pryv.sh 1.6`

9. Start mongo service
10. Restore Mongodump    
    `cp -r /var/pryv1.5/pryv/mongodb/backup/pryv-node/ /var/pryv/pryv/mongodb/backup/2021-06-02-pryv-node`
    `docker exec -ti pryvio_mongodb /app/bin/mongodb/bin/mongorestore --drop -d pryv-node /app/backup/2021-06-02-pryv-node`
11. Start all services

In case of failure

1. Stop all services
2. Move `/var/pryv` to `/var/pryv2`
3. Move `/var/pryv1` to `/var/pryv`
4. Start all services
5.   
6.   
7.   
    
Restore Redis

1. Stop NGNIX
   1. `docker stop pryvio_nginx`
2. Backup Redis    
   1. `docker exec -ti pryvio_redis bash`
   2. `npm install redis-dump -g`
   3. `redis-dump --json > /app/data/redis1.6-dump.json`
   4. `cd /app/data/`
   5. `cat redis1.6-dump.json | redis-dump --convert > redis1.6-dump.cmds `
3. Stop Redis container
   1. `docker stop pryvio_redis`
4. Generate backups: /var/pryv1.6/redis-bkp/
   1. `cp /var/pryv1.6/pryv/redis/data/* /var/pryv1.6/redis-bkp/1.6`
   2. `cp /var/pryv1.5/pryv/redis/data/* /var/pryv1.6/redis-bkp/1.5`
5. Replace redis1.6 by 1.5 data
   1. `cp /var/pryv1.5/pryv/redis/data/* /var/pryv1.6/pryv/redis/data/ `
4. Restart ONLY Redis container
   1. `PRYV_CONF_ROOT="/var/pryv" docker-compose -f pryv/pryv.yml start redis`
5. Restore 1.6 content
   1. `docker exec -ti pryvio_redis bash`
   2. `cd /app/data`
   3. `cat redis1.6-dump.cmds | redis-cli`
6. Restart Pryv.io 
   1. `/var/pryv/run-pryv`
