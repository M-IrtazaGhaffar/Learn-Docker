## Learn-Docker

# Docker files and compose
Use Docker file (Dockerfile) to give the specifications of the docker Image while building. All of the commands to run the docker Image is used in this container.

# Docker Compose
Use Docker Compose (docker-compose.yml) to specify the number of containers which are running together to perform certain operations. All of the containers are performing the given specifications written in the file.

# Docker Networking
Docker uses 3 types of Networking to connect the Containers with the Internet.
1). None: 
      No connection is being established.
2). Bridge (Default): 
      Docker establishes a bridge between the conatiner and the internet for connection.
3). Host: s
      Docker uses Host Network as a local matchine not like a container which is attached to internet with a bridge. We also don't have to do the port mapping because it is being used as a Local Matchine not a container aswell.

Check all the Images using types of network
```
docker network ls
```

Run a Docker with Custom Network
```
docker run -it --network=host redis
```

# Commands

Run Docker
```
docker run redis
```

Run Docker in Interactive Mode
```
docker run -it redis
```

Run Docker in Background
```
docker run redis -d
```

Run Docker and give a Specific Name to Container
```
docker run --name RedisContainer redis
```

Run Docker with Port Mapping (Local:Docker)
```
docker run -p 6379:6379 redis
```

Run Docker with ENV
```
docker run -e PORT=12345 redis
```

Search in Docker
```
docker search YourSearchName
```

Check Running Images in Docker
```
docker ps
```
Check All Images in Docker
```
docker ps -a
```

Stop a Container in Docker
```
docker stop ImageID
```

Restart an Image in Docker
```
docker restart ImageID
```

Build Docker Image
```
docker build -t ImageName
```

Check a Running Container in Docker
```
docker exec redis
```
