<h1 style="text-align: center"> BYOND ASSESMENT</h1>

## Description
This repository was created as an assesment task provided by BYOND. This version uses the following files:

````bash
.
├── Dockerfile
├── JenkinsFile
├── README.md
├── app.py
├── docker-compose.yml
├── requirements.txt
└── variables.txt
````

## File Description
- DockerFile: Required for creating the docker image using the code inside app.py.
- docker-compose.yml: Required for starting docker services.
- JenkinsFile: Required for downloading the code from this repository, generating the image and starting de docker-composes services.
- app.py: Main application source file coded in python.
- requirements.txt: Requested dependencies for app.py application.
- variables.txt: Variable files for application ports.

## Usage
### Dockerfile
This file uses the lastest available python image from DockerHub. Is required by the JenkinsFile to generate the Docker image with the application. 

### Jenkins 
To execute the JenkinsFile you will require Docker and GitHub plugins installed and configured in Jenkins.

````groovy
docker.build("byond:${env.BUILD_ID}").tag("latest")
````

The images are tags using the BUILD_ID variable and the lastest tag after is regenerated. The previews images are not deleted. Once the image is build and tagged, the pipeline will start the lastest one by docker-compose.

````groovy
stage('Deploy new Image') {
            steps {
                script {
                    sh "docker-compose --env-file variables.txt down"
                    sh "docker-compose --env-file variables.txt up -d --force-recreate"
                }
            }
        }
````

### Docker compose configuration

#### **Redis Volume**

````yaml
volumes:
    redis:
services:
    redis_server:
        image: redis
        restart: always
        volumes:     
            - redis:/data
````

For redis service we need to create a volume. 

````bash
docker volume create redis
````

After that we can verify the creation with the following command

````bash
docker volume inspect redis
````

````json
[
    {
        "CreatedAt": "2021-05-15T18:46:50Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/redis/_data",
        "Name": "redis",
        "Options": {},
        "Scope": "local"
    }
]
````

#### **Ports**

Is required by this file to define de following variables in the variables.txt 

````properties
CONTAINER_PORT=8080
HOST_PORT=5000
````

This variables will allow you to change the desired ports for app.py service if needed. 

````yaml
python_server:
        image: byond:latest
        restart: always
        ports:
            - ${HOST_PORT}:${CONTAINER_PORT}
        environment:
            REDIS_HOST: redis_server
            REDIS_PORT: 6379
            BIND_PORT: ${CONTAINER_PORT}
        depends_on:
            - redis_server
````

You can also change the image tag if necesary. To achieve it, you should change the lastest tag to the preview image you currently have.

````bash
REPOSITORY     TAG       IMAGE ID       CREATED             SIZE
byond          13       fa606625a420   About an hour ago   841MB
byond          14       fa606625a420   About an hour ago   841MB
byond          15       fa606625a420   About an hour ago   841MB
byond          16       fa606625a420   About an hour ago   841MB
byond          latest   fa606625a420   About an hour ago   841MB
````

````yaml
python_server:
        image: byond:16
        restart: always
````

#### **Resources**
Resources are limited to 200 milicores and 256 mb of memory

````yaml
deploy:
    resources:
        limits:
            cpus: '0.20'
            memory: 512M
````

#### **HealthChecks**

````yaml
healthcheck:
    test: ["CMD", "curl", "-f", "http://python_server:${CONTAINER_PORT}/"]
    interval: ${INTERVAL}
    timeout: ${TIMEOUT}
    retries: ${RETRIES}
````


Endpoint, interval, timeout and retries values are set as variables in the variables files. 

````properties
ENDPOINT=/
TIMEOUT=10s
INTERVAL=5s
RETRIES=3
````

Once you changed the file. You need to restart the services using docker-compose command or redeploying the services by executing the JenkinsFile.

````bash
docker-compose --env-file variables.txt stop
Stopping byond_python_server_1 ... done
Stopping byond_redis_server_1  ... done

docker-compose --env-file variables.txt up -d
Starting byond_redis_server_1 ... done
Starting byond_python_server_1 ... done
````