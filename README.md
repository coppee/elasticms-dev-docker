#Development environment for elasticms
##About this repository
This project contains a ready to use elasticms environment for development purposes with the following stack:

- elasticsearch (the heart of the solution)
- kibana (to see what happening in the heart)
- traefik (as middelware)
- postgres (as RDBMS)
- mariadb (as RDMBS in low cost infrastructure)
- sqlite (as "RDBMS" for portable, and single user, use)
- Apache Tika (to extract asset's contents)
- minio (as file storage, compatible with the AWS S3 API)
- redis (to store and share PHP session)
- elasticms (the ems content management application)
- skeleton (the ems content delivery application)

To work with the elastic stack version you want, from 5 to 7, open a console in one of the following folders:
- elastic5
- elastic7

If you want to switch from one version to the other execute ```docker-compose down``` before changing working directory. In order to ensure there is no conflict in processes name.

---
**NOTE**

The command ```docker-compose down``` won't delete persisted data (i.e. database's data) in Docker's volumes, nevertheless if you switch from an elastic stack version to another be aware Dockers volumes are not share between docker-compose projects. You'll have to recreate and reindex your content. You may want to mount local folders instead of Docker volumes. I.e. for Postgres you can change the line ```- postgres:/var/lib/postgresql/data``` by ``- ../databases/postgres:/var/lib/postgresql/data``. If so::
- Don't do that for elasticsearch data, they are not compatible from one version to the other
- If something as change in a datasource consider to reindex it elasticsearch


---

#Requirements
In order to have a working elasticsearch cluster you must have at least [4GB dedicated to you docker environment](https://github.com/elastic/elasticsearch/issues/51196). You might also want to check those [production recommendations](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-prod-prerequisites).

##Baby step
###Launch docker-compose
The first thing to do is to start your environment:

```docker-compose up -d```

You can follow that everything is starting smoothly with:

```docker-compose logs -f```

Once it's started check all is working:

```docker-compose ps```

Check the image's versions with the following command: ```docker-compose images```.

You might have notice that there are 3 instances of the elasticms: ems_mysql, ems_pgsql and ems_sqlite. The reason is that Symfony (the PHP framework behind elasticms), for performance reasons, generates cache files specific to the DB driver used. So you can't use the same instance of elasticms with different RDBMS. 


###Check elasticsearch cluster's health
Go to the [Kibana dev console](http://kibana.localhost/app/dev_tools#/console) and check the cluster health:
```
GET _cluster/health
```

You should see something like:
```json
{
  "cluster_name" : "es-docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 6,
  "active_shards" : 12,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```
Note that you can also check cluster's health from the command line: ```docker-compose exec es01 curl http://localhost:9200/_cluster/health```

If the Kibana url is not working you should check that its route has been correctly registered in [Traefik](http://localhost:8888/dashboard/#/http/routers). On that page you should see those host rules:
- kibana.localhost
- tika.localhost
- minio.localhost
- es.localhost
- demo-admin.localhost

###Initiate databases
Here we will just initiate the database and the user. The database schema will be initiated later with the Symfony console. 
 
####Postgres
To initiate a postgres DB run ```../init_pgsql.sh demo``` or you can launch those commands:

```
docker-compose exec -e PGUSER=postgres -e PGPASSWORD=adminpg -T postgres psql -c "CREATE DATABASE demo;"
docker-compose exec -e PGUSER=postgres -e PGPASSWORD=adminpg -T postgres psql -c "CREATE USER demo WITH ENCRYPTED PASSWORD 'demo';"
docker-compose exec -e PGUSER=postgres -e PGPASSWORD=adminpg -T postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE demo TO demo;"
```

You can use the ``../drop_pgsql.sh demo`` to drop the database.

####MySQL
To initiate a postgres DB run ```../init_mysql.sh demo``` or you can launch those commands:

```
docker-compose exec mariadb mysql --user=root --password=mariadb -e "CREATE DATABASE IF NOT EXISTS demo;"
docker-compose exec mariadb mysql --user=root --password=mariadb -e "CREATE USER demo@'%' IDENTIFIED BY 'demo';"
docker-compose exec mariadb mysql --user=root --password=mariadb -e "GRANT ALL PRIVILEGES ON demo.* TO demo@'%';"
docker-compose exec mariadb mysql --user=root --password=mariadb -e "CREATE USER demo@'localhost' IDENTIFIED BY 'demo';"
docker-compose exec mariadb mysql --user=root --password=mariadb -e "GRANT ALL PRIVILEGES ON demo.* TO demo@'localhost';"
docker-compose exec mariadb mysql --user=root --password=mariadb -e "show databases;"
```
You can use the ``../drop_mysql.sh demo`` to drop the database.

###SQLite
There is nothing to do at this time. A demo.db file has been already created in the ``databases`` folder by the elasticms boot script.

###Other RDBMS
There is currently no support for other RDBMS, but if the RDBMS considered is currently [supported by doctrine](https://www.doctrine-project.org/projects/doctrine-dbal/en/2.10/reference/platforms.html) you will be able to easily generate the database schema as well. So up to you to use the database platform you want.      

##Instantiate the database's schema
To initialize an elasticms schema we will use the Symfony console to execute the doctrine migration scripts. In order to access to the Symfony console we will execute a bash in the elasticms processes with the following command:
```docker-compose exec ems_pgsql bash```, ```docker-compose exec ems_mysql bash``` or ```docker-compose exec ems_sqlite bash```.

    Once there, you can call the Demo's Symfony console : ```demo```. This will list all available elasticms's commands. To run the migration scripts: ```demo doctrine:migrations:migrate```.

Another option is to recreate the elasticms docker process: ```docker-compose up -d --force-recreate ems_pgsql```, as the elasticms docker image starting script is executing the doctrine migration scripts on its own.  

You should now be able to show the elasticms [login window](http://demo-admin.localhost). For that you need to [create an admin account](#Create a user). You can see that everything looks good by checking the [elasticms status page](http://demo-admin.localhost/status).

##About the Symfony console
In the ``configs`` folder there are 4 folders:
- ems-pgsql
- ems-mysql
- ems-sqlite
- skeleton

You can create as many Dotenv files as you want in those folders. Per folder a virtual host will be setup for the domains specified by the variables ``SERVER_NAME`` and ``SERVER_ALIASES``. For each domain you defined you have to add in Traefik via the docker's label in the corresponding process definition:

```yaml
  ems_pgsql:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.demo-admin.rule=Host(`demo-admin.localhost`)"
      - "traefik.http.routers.demo-admin.entrypoints=web"
      - "traefik.http.routers.demo-pgsql-admin.rule=Host(`demo-pgsql-admin.localhost`)"
      - "traefik.http.routers.demo-pgsql-admin.entrypoints=web"
      - "traefik.http.routers.demo-admin-dev.rule=Host(`demo-admin-dev.localhost`)"
      - "traefik.http.routers.demo-admin-dev.entrypoints=web"
```
 When you update a Dotenv file you have to recreate the docker-compose process: ```docker-compose up -d --force-recreate ems_pgsql```.
 
 So an elasticms pgsql docker-compose process can be used by as many ems projects as you want. Until they are all using a Postgres database in this case.
 
 It's also important to interact with those projects via the Symfony console, not only via urls. To do so, the elasticms docker's image creates one shell scripts per Dotenv files within the elasticms's docker process in the ``/opt/bin`` folder. Those scripts have being named from the basename of the corresponding Dotenv file: ``demo.env`` => ```/opt/bin/demo```. 
 Then, you can call the Symfony console ```/opt/bin/demo``` from a bash inside the docker process ```docker-compose exec ems_pgsql bash```. Or directly from your host: ```docker-compose exec ems_pgsql /opt/bin/demo```. Finally, as the folder ``/opt/bin`` is in the path, ``docker-compose exec ems_pgsql demo`` usually works.

##Create a user
Execute this command ``docker-compose exec ems_pgsql demo fos:user:create --super-admin`` and answer to the questions. You are now able to login [elasticms](http://demo-admin.localhost).

##Create a minio demo bucket
Go to the [minio interface](http://minio.localhost/minio/login) and login with the credentials:
- Access key: ``accesskey``
- Secret key: ``secretkey``
In the bottom-right corner click on the ``+`` button and select  ``Create bucket``. Name it ``demo``. 

##Configure your content
1. Define the publication environments
2. Define the content types (encoding forms and mapping)

##Load the demo website
```
#Load the sample SQL dump
docker-compose exec ems_pgsql demo sql --file=/opt/samples/demo.sql
#List publication environments
docker-compose exec ems_pgsql demo ems:environment:list
#Index environments
docker-compose exec ems_pgsql demo ems:environment:rebuild preview
docker-compose exec ems_pgsql demo ems:environment:rebuild template
docker-compose exec ems_pgsql demo ems:environment:rebuild live
```

##Developments

###Debug emails
You can check sent emails with [MailHog](http://mailhog.localhost/#).

##To dos
- Remove the sqlite hotfix framework.yaml file (add Redis support and parametrized the session handler)
- Load the skeleton skin archive
- Skeleton
- Explain how to work with a bundle git repository