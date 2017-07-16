# Docker Cheat Sheet

## Docker

Docker is a software container platform and this is designed to make it easier to create, deploy, and run applications by using containers. This tool is designed to benefit both developers and system administrators, making it a part of many DevOps (developers + operations) toolchains and used by developers to eliminate “works on my machine” problems when collaborating on code with co-workers. Operators use it to run and manage apps side by side in isolated containers. 

## Containers

A container is an instance of an image and pull only downloads the image, it doesn't create a container. 

### Executing Commands

* [`docker exec`](https://docs.docker.com/engine/reference/commandline/exec) to execute a command in container.

To enter a running container, attach a new shell process to a running container called foo, use: `docker exec -it foo /bin/bash`.

## Images

An image is an inert, immutable, file that’s a snapshot of a container. Images are created with the build command, and they’ll produce a container when started with run and these are stored in a Docker registry such as registry.hub.docker.com. 

### Cleaning up

While you can use the `docker rmi` command to remove specific images, there's a tool called [docker-gc](https://github.com/spotify/docker-gc) that will clean up images that are no longer used by any containers in a safe manner.

### Load/Save image

Load an image from file:
```
docker load < my_image.tar.gz
```

Save an existing image:
```
docker save my_image:my_tag | gzip > my_image.tar.gz
```

### Import/Export container

Import a container as an image from file:
```
cat my_container.tar.gz | docker import - my_image:my_tag
```

Export an existing container:
```
docker export my_container | gzip > my_container.tar.gz
```

## Networks

Docker has a [networks](https://docs.docker.com/engine/userguide/networking/) feature. 

## Dockerfile

[The configuration file](https://docs.docker.com/engine/reference/builder/). Sets up a Docker container when you run `docker build` on it. Vastly preferable to `docker commit`.  

## Tips

Sources:

* [15 Docker Tips in 5 minutes](http://sssslide.com/speakerdeck.com/bmorearty/15-docker-tips-in-5-minutes)
* [CodeFresh Everyday Hacks Docker](https://codefresh.io/blog/everyday-hacks-docker/)

### Set Up

`docker pull ubuntu` pulls a base image.

`alias dl='docker ps -l -q'` to get the id of the last-run container, you can set alias.

### Create a container 

`docker run -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"`

### Stop a container 

~~~
docker stop `dl` 
~~~

### Start a container
~~~
docker start `dl` 
~~~

### Restart a container 
~~~
docker restart `dl` 
~~~

### Connect to a running Container
~~~
docker attach `dl` 
~~~

### Copy file in a Container to the host

`docker cp `dl`:/etc/passwd`

### Mount the directory in host to a Container 

`docker run -v /home/vagrant/test:/root/test ubuntu echo yo`

### Delete a Container.
~~~
dockr rm `dl` 
~~~

### Show running Containers; where -a option shows running and stopped Containers.

`docker ps`

### Show Container information like IP adress
~~~
docker inspect `dl` 
~~~

### Show log of a Container
~~~
docker logs `dl` 
~~~

### Show running process in a Container
~~~
docker top `dl` 
~~~

### Create a image from a Container

`docker run -d ubuntu /bin/sh -c "apt-get install -y hello"
docker commit -m "My first container" `dl` test/hello`

### Create a image with Dockerfile

`echo -e "FROM base\nRUN apt-get install hello\nCMD hello" > Dockerfile
docker build tcnksm/hello .`

### login to a image 

`docker run -rm -t -i test/hello /bin/bash`

### Push a imges to remote repository.  

`docker login
docker push test/hello`

### Delete a image

`docker rmi test/hello` 

### Show all images

`docker images`

### Show image information like IP adress

`docker inspect test/hello`

### Show command history of a image

`docker history test/hello`

### df 

`docker system df` presents a summary of the space currently used by different docker objects.

### Heredoc Docker Container

```
docker build -t htop - << EOF
FROM alpine
RUN apk --no-cache add htop
EOF
```

### Last Ids

```
alias dl='docker ps -l -q'
docker run ubuntu echo hello world
docker commit $(dl) helloworld
```

### Commit with command (needs Dockerfile)

```
docker commit -run='{"Cmd":["postgres", "-too -many -opts"]}' $(dl) postgres
```

### Get IP address

```
docker inspect $(dl) | grep IPAddress | cut -d '"' -f 4
```

or install [jq](https://stedolan.github.io/jq/):

```
docker inspect $(dl) | jq -r '.[0].NetworkSettings.IPAddress'
```

or using a [go template](https://docs.docker.com/engine/reference/commandline/inspect):

```
docker inspect -f '{{ .NetworkSettings.IPAddress }}' <container_name>
```

or when building an image from Dockerfile, when you want to pass in a build argument:

```
DOCKER_HOST_IP=`ifconfig | grep -E "([0-9]{1,3}\.){3}[0-9]{1,3}" | grep -v 127.0.0.1 | awk '{ print $2 }' | cut -f2 -d: | head -n1`
echo DOCKER_HOST_IP = $DOCKER_HOST_IP
docker build \
  --build-arg ARTIFACTORY_ADDRESS=$DOCKER_HOST_IP 
  -t sometag \
  some-directory/
 ```

### Get port mapping

```
docker inspect -f '{{range $p, $conf := .NetworkSettings.Ports}} {{$p}} -> {{(index $conf 0).HostPort}} {{end}}' <containername>
```

### Find containers by regular expression

```
for i in $(docker ps -a | grep "REGEXP_PATTERN" | cut -f1 -d" "); do echo $i; done
```

### Get Environment Settings

```
docker run --rm ubuntu env
```

### Kill running containers

```
docker kill $(docker ps -q)
```

### Delete old containers

```
docker ps -a | grep 'weeks ago' | awk '{print $1}' | xargs docker rm
```

### Delete stopped containers

```
docker rm -v $(docker ps -a -q -f status=exited)
```

### Delete dangling images

```
docker rmi $(docker images -q -f dangling=true)
```

### Delete all images

```
docker rmi $(docker images -q)
```

### Delete dangling volumes

As of Docker 1.9:

```
docker volume rm $(docker volume ls -q -f dangling=true)
```

In 1.9.0, the filter `dangling=false` does _not_ work - it is ignored and will list all volumes.

### Show image dependencies

```
docker images -viz | dot -Tpng -o docker.png
```

### Slimming down Docker containers

- Cleaning APT in a RUN layer

This should be done in the same layer as other apt commands.
Otherwise, the previous layers still persist the original information and your images will still be fat.

```
RUN {apt commands} \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

- Flatten an image
```
ID=$(docker run -d image-name /bin/bash)
docker export $ID | docker import – flat-image-name
```

- For backup
```
ID=$(docker run -d image-name /bin/bash)
(docker export $ID | gzip -c > image.tgz)
gzip -dc image.tgz | docker import - flat-image-name
```

### Monitor system resource utilization for running containers

To check the CPU, memory, and network I/O usage of a single container, you can use:

```
docker stats <container>
```

For all containers listed by id:

```
docker stats $(docker ps -q)
```

For all containers listed by name:

```
docker stats $(docker ps --format '{{.Names}}')
```

For all containers listed by image:

```
docker ps -a -f ancestor=ubuntu
```

Remove all untagged images
```
docker rmi $(docker images | grep “^” | awk “{print $3}”)
```

Remove container by a regular expression
```
docker ps -a | grep wildfly | awk '{print $1}' | xargs docker rm -f
```
Remove all exited containers
```
docker rm -f $(docker ps -a | grep Exit | awk '{ print $1 }')
```

