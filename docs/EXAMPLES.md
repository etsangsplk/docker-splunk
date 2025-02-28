## Examples

The purpose of this section is to showcase a wide variety of examples on how the `docker-splunk` project can be used. 

Note that for more complex scenarios, we will opt to use a [Docker compose file](https://docs.docker.com/compose/compose-file/) instead of the CLI for the sake of readability.

## I want to...

* [Create a standalone](#create-standalone-from-cli)
    * [...with the CLI](#create-standalone-from-cli)
    * [...with a compose file](#create-standalone-from-compose)
    * [...with a Splunk license](#create-standalone-with-license)
    * [...with HEC](#create-standalone-with-hec)
    * [...with any app](#create-standalone-with-app)
    * [...with a SplunkBase app](#create-standalone-with-splunkbase-app)
* [Create standalone and universal forwarder](#create-standalone-and-universal-forwarder)
* [Create heavy forwarder](#create-heavy-forwarder)
* [Create heavy forwarder and deployment server](#create-heavy-forwarder-and-deployment-server)
* [Create indexer cluster](#create-indexer-cluster)
* [Create search head cluster](#create-search-head-cluster)
* [Create indexer cluster and search head cluster](#create-indexer-cluster-and-search-head-cluster)
* [Enable root endpoint on SplunkWeb](#enable-root-endpoint-on-splunkweb)
* [More](#more)

## Create standalone from CLI
Execute the following to bring up your deployment:
```
$ docker run --name so1 --hostname so1 -p 8000:8000 -e "SPLUNK_PASSWORD=<password>" -e "SPLUNK_START_ARGS=--accept-license" -it splunk/splunk:latest
```

## Create standalone from compose
<details><summary>docker-compose.yml</summary><p>

```
version: "3.6"

services:
  so1:
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    container_name: so1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_PASSWORD
    ports:
      - 8000
```
</p></details>

Execute the following to bring up your deployment:
```
$ SPLUNK_PASSWORD=<password> docker-compose up -d
```

## Create standalone with license
Adding a Splunk Enterprise license can be done in multiple ways. Please review the following compose files below to see how it can be achieved, either with a license hosted on a webserver or with a license file as a direct mount.

<details><summary>docker-compose.yml - license from URL</summary><p>

```
version: "3.6"

services:
  so1:
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    container_name: so1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_LICENSE_URI=http://company.com/path/to/splunk.lic
      - SPLUNK_PASSWORD
    ports:
      - 8000
```
</p></details>

<details><summary>docker-compose.yml - license from file</summary><p>

```
version: "3.6"

services:
  so1:
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    container_name: so1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_LICENSE_URI=/tmp/license/splunk.lic
      - SPLUNK_PASSWORD
    ports:
      - 8000
    volumes:
      - ./splunk.lic:/tmp/license/splunk.lic
```
</p></details>


Execute the following to bring up your deployment:
```
$ SPLUNK_PASSWORD=<password> docker-compose up -d
```

## Create standalone with HEC
To learn more about what the HTTP event collector (HEC) is and how to use it, please review the documentation [here](https://docs.splunk.com/Documentation/Splunk/latest/Data/UsetheHTTPEventCollector).

<details><summary>docker-compose.yml</summary><p>

```
version: "3.6"

services:
  so1:
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    container_name: so1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_HEC_TOKEN=abcd1234
      - SPLUNK_PASSWORD
    ports:
      - 8000
```
</p></details>

Execute the following to bring up your deployment:
```
$ SPLUNK_PASSWORD=<password> docker-compose up -d
```

To validate HEC is provisioned properly and functional:
```
$ curl -k https://localhost:8088/services/collector/event -H "Authorization: Splunk abcd1234" -d '{"event": "hello world"}'
{"text": "Success", "code": 0}
```

## Create standalone with app
<details><summary>docker-compose.yml</summary><p>

```
version: "3.6"

services:
  so1:
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    container_name: so1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_APPS_URL=http://company.com/path/to/app.tgz
      - SPLUNK_PASSWORD
    ports:
      - 8000
```
</p></details>

Execute the following to bring up your deployment:
```
$ SPLUNK_PASSWORD=<password> docker-compose up -d
```

## Create standalone with SplunkBase app
<details><summary>docker-compose.yml</summary><p>

```
version: "3.6"

services:
  so1:
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    container_name: so1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_APPS_URL=https://splunkbase.splunk.com/app/2890/release/4.1.0/download
      - SPLUNKBASE_USERNAME=<username>
      - SPLUNKBASE_PASSWORD
      - SPLUNK_PASSWORD
    ports:
      - 8000
```
</p></details>

Execute the following to bring up your deployment:
```
$ SPLUNKBASE_PASSWORD=<splunkbase_password> SPLUNK_PASSWORD=<password> docker-compose up -d
```

## Create standalone and universal forwarder
<details><summary>docker-compose.yml</summary><p>

```
version: "3.6"

networks:
  splunknet:
    driver: bridge
    attachable: true

services:
  uf1:
    networks:
      splunknet:
        aliases:
          - uf1
    image: ${UF_IMAGE:-splunk/universalforwarder:latest}
    hostname: uf1
    container_name: uf1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_STANDALONE_URL=so1
      - SPLUNK_ADD=udp 1514,monitor /var/log/*
      - SPLUNK_PASSWORD
    ports:
      - 8089

  so1:
    networks:
      splunknet:
        aliases:
          - so1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: so1
    container_name: so1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_STANDALONE_URL=so1
      - SPLUNK_PASSWORD
    ports:
      - 8000
      - 8089
```
</p></details>

Execute the following to bring up your deployment:
```
$ SPLUNK_PASSWORD=<password> docker-compose up -d
```

## Create heavy forwarder
The following will allow you spin up a forwarder, and stream its logs to an independent, external indexer located at `idx1-splunk.company.internal`, as long as that hostname is reachable on your network.

<details><summary>docker-compose.yml</summary><p>

```
version: "3.6"

networks:
  splunknet:
    driver: bridge
    attachable: true

services:
  hf1:
    networks:
      splunknet:
        aliases:
          - hf1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: hf1
    container_name: hf1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_ROLE=splunk_heavy_forwarder
      - SPLUNK_INDEXER_URL=idx1-splunk.company.internal
      - SPLUNK_ADD=tcp 1514
      - SPLUNK_PASSWORD
    ports:
      - 1514
```
</p></details>

Execute the following to bring up your deployment:
```
$ SPLUNK_PASSWORD=<password> docker-compose up -d
```

## Create heavy forwarder and deployment server
The following will allow you spin up a forwarder, and stream its logs to an independent, external indexer located at `idx1-splunk.company.internal`, as long as that hostname is reachable on your network. Additionally, it brings up a deployment server, which will download an app and distribute it to the heavy forwarder.

<details><summary>docker-compose.yml</summary><p>

```
version: "3.6"

networks:
  splunknet:
    driver: bridge
    attachable: true

services:
  hf1:
    networks:
      splunknet:
        aliases:
          - hf1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: hf1
    container_name: hf1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_ROLE=splunk_heavy_forwarder
      - SPLUNK_INDEXER_URL=idx1-splunk.company.internal
      - SPLUNK_DEPLOYMENT_SERVER=depserver1
      - SPLUNK_ADD=tcp 1514
      - SPLUNK_PASSWORD
    ports:
      - 1514
  
  depserver1:
    networks:
      splunknet:
        aliases:
          - depserver1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: depserver1
    container_name: depserver1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_ROLE=splunk_deployment_server
      - SPLUNK_APPS_URL=https://artifact.company.internal/splunk_app.tgz
      - SPLUNK_PASSWORD
```
</p></details>

Execute the following to bring up your deployment:
```
$ SPLUNK_PASSWORD=<password> docker-compose up -d
```

## Create indexer cluster
To enable indexer cluster, we'll need to generate some common passwords and secret keys across all members of the deployment. To facilitate this, you can use the `splunk/splunk` image with the `create-defaults` command as so:
```
$ docker run -it -e SPLUNK_PASSWORD=<password> splunk/splunk:latest create-defaults > default.yml
```

Additionally, review the `docker-compose.yml` below to understand how linking Splunk instances together through roles and environment variables is accomplished:
<details><summary>docker-compose.yml</summary><p>

```
version: "3.6"

networks:
  splunknet:
    driver: bridge
    attachable: true

services:
  sh1:
    networks:
      splunknet:
        aliases:
          - sh1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: sh1
    container_name: sh1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_search_head
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  cm1:
    networks:
      splunknet:
        aliases:
          - cm1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    command: start
    hostname: cm1
    container_name: cm1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_cluster_master
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  idx1:
    networks:
      splunknet:
        aliases:
          - idx1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    command: start
    hostname: idx1
    container_name: idx1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_indexer
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  idx2:
    networks:
      splunknet:
        aliases:
          - idx2
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    command: start
    hostname: idx2
    container_name: idx2
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_indexer
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  idx3:
    networks:
      splunknet:
        aliases:
          - idx3
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    command: start
    hostname: idx3
    container_name: idx3
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_indexer
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml
```
</p></details>

Execute the following to bring up your deployment:
```
$ SPLUNK_PASSWORD=<password> docker-compose up -d
```

## Create search head cluster
To enable search head clustering, we'll need to generate some common passwords and secret keys across all members of the deployment. To facilitate this, you can use the `splunk/splunk` image with the `create-defaults` command as so:
```
$ docker run -it -e SPLUNK_PASSWORD=<password> splunk/splunk:latest create-defaults > default.yml
```

Additionally, review the `docker-compose.yml` below to understand how linking Splunk instances together through roles and environment variables is accomplished:
<details><summary>docker-compose.yml</summary><p>

```
version: "3.6"

networks:
  splunknet:
    driver: bridge
    attachable: true

services:
  sh1:
    networks:
      splunknet:
        aliases:
          - sh1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: sh1
    container_name: sh1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_ROLE=splunk_search_head_captain
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  sh2:
    networks:
      splunknet:
        aliases:
          - sh2
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: sh2
    container_name: sh2
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_ROLE=splunk_search_head
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  sh3:
    networks:
      splunknet:
        aliases:
          - sh3
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: sh3
    container_name: sh3
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_ROLE=splunk_search_head
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  dep1:
    networks:
      splunknet:
        aliases:
          - dep1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: dep1
    container_name: dep1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_ROLE=splunk_deployer
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  idx1:
    networks:
      splunknet:
        aliases:
          - idx1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: idx1
    container_name: idx1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_ROLE=splunk_indexer
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml
```
</p></details>

Execute the following to bring up your deployment:
```
$ docker-compose up -d
```

## Create indexer cluster and search head cluster
To enable both clustering modes, we'll need to generate some common passwords and secret keys across all members of the deployment. To facilitate this, you can use the `splunk/splunk` image with the `create-defaults` command as so:
```
$ docker run -it -e SPLUNK_PASSWORD=<password> splunk/splunk:latest create-defaults > default.yml
```

Additionally, review the `docker-compose.yml` below to understand how linking Splunk instances together through roles and environment variables is accomplished:
<details><summary>docker-compose.yml</summary><p>

```
version: "3.6"

networks:
  splunknet:
    driver: bridge
    attachable: true

services:
  sh1:
    networks:
      splunknet:
        aliases:
          - sh1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: sh1
    container_name: sh1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_search_head_captain
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  sh2:
    networks:
      splunknet:
        aliases:
          - sh2
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: sh2
    container_name: sh2
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_search_head
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  sh3:
    networks:
      splunknet:
        aliases:
          - sh3
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: sh3
    container_name: sh3
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_search_head
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  dep1:
    networks:
      splunknet:
        aliases:
          - dep1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: dep1
    container_name: dep1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_deployer
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  cm1:
    networks:
      splunknet:
        aliases:
          - cm1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: cm1
    container_name: cm1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_cluster_master
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  idx1:
    networks:
      splunknet:
        aliases:
          - idx1
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: idx1
    container_name: idx1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_indexer
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  idx2:
    networks:
      splunknet:
        aliases:
          - idx2
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: idx2
    container_name: idx2
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_indexer
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml

  idx3:
    networks:
      splunknet:
        aliases:
          - idx3
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    hostname: idx3
    container_name: idx3
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_INDEXER_URL=idx1,idx2,idx3
      - SPLUNK_SEARCH_HEAD_URL=sh2,sh3
      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=sh1
      - SPLUNK_CLUSTER_MASTER_URL=cm1
      - SPLUNK_ROLE=splunk_indexer
      - SPLUNK_DEPLOYER_URL=dep1
    ports:
      - 8000
      - 8089
    volumes:
      - ./default.yml:/tmp/defaults/default.yml
```
</p></details>

Execute the following to bring up your deployment:
```
$ docker-compose up -d
```

## Enable root endpoint on SplunkWeb
<details><summary>docker-compose.yml</summary><p>

```
version: "3.6"

services:
  so1:
    image: ${SPLUNK_IMAGE:-splunk/splunk:latest}
    container_name: so1
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_ROOT_ENDPOINT=/splunkweb
      - SPLUNK_PASSWORD
    ports:
      - 8000
```
</p></details>

Execute the following to bring up your deployment:
```
$ SPLUNK_PASSWORD=<password> docker-compose up -d
```

Then, visit SplunkWeb on your browser with the root endpoint in the URL, such as `http://localhost:8000/splunkweb`.

## More
There are a variety of Docker compose scenarios in the `docker-splunk` repo [here](https://github.com/splunk/docker-splunk/tree/develop/test_scenarios). Please feel free to use any of those for reference in terms of different topologies!
