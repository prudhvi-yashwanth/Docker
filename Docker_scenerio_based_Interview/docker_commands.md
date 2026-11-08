The first thing to check on any machine is whether the client can reach the daemon:
docker version
docker info
For a fuller picture of the engine's state, use:
docker version - show client and server versions
docker info - show engine state and totals
docker run hello-world - pull and run the test image end to end
docker run IMAGE - create and start a container from an image
docker run IMAGE command args - override the default command
docker ps - list running containers
docker ps -a - list all containers, including stopped ones
docker run -d IMAGE - run a container in the background
docker run --name NAME IMAGE - give the container a chosen name
docker stop NAME - stop a running container gracefully
docker start NAME - start a stopped container again
docker rm NAME - remove a stopped container
docker rm -f NAME - force-remove a running container
docker run --rm ... - auto-remove the container when it exits
docker pull IMAGE - download an image without running it
docker inspect nginx
docker images - list images stored locally
docker inspect IMAGE - show full details of an image as JSON

Dangling Docker images are old or intermediate layers that have no tag (<none>:<none>) and are not referenced by any running or stopped container
docker image prune
docker rmi redis
docker run -it IMAGE sh - start a container with an interactive shell
-i and -t - keep input open and allocate a terminaldocker run -it IMAGE sh - start a container with an interactive shell
-i and -t - keep input open and allocate a terminal
docker exec CONTAINER command - run a command in a running container
docker exec -it CONTAINER sh - open a shell in a running container

docker exec exec-lab sh -c 'echo "exec reached the container" > /tmp/deploy-check.txt'
docker exec exec-lab cat /tmp/deploy-check.txt


docker logs CONTAINER - show a container's output
docker logs -f CONTAINER - follow the log stream live
docker logs --tail N CONTAINER - show only the last N lines
docker logs --since TIME CONTAINER - show only recent lines

docker cp CONTAINER:/path ./local - copy a file out of a container
docker cp ./local CONTAINER:/path - copy a file into a container

docker pause CONTAINER / docker unpause CONTAINER - freeze and resume a container
When you run docker pause, the memory contents are fully preserved (retained), and the container’s network state—including the Forwarding Database (FDB) entries and VXLAN configurations—remains intact and is not lost


docker inspect CONTAINER - full container details as JSON
docker inspect -f '{{.State.Status}}' CONTAINER - extract one field with a format template
docker inspect -f '{{.State.Status}}' web
docker inspect -f '{{.State.ExitCode}}' web
docker inspect -f '{{.State.Running}}' web
docker run --restart POLICY ... - set a restart policy (no, on-failure, always, unless-stopped)
There are four policies:

no - the default. Docker never restarts the container. When it exits, it stays exited.
on-failure - restart only if the process exits with a non-zero code. A clean exit (code 0) is left alone. You can cap the attempts with on-failure:5.
always - restart whenever it stops, for any reason, and also start it again when the Docker daemon itself restarts.
unless-stopped - like always, but if you deliberately docker stop it, Docker does not bring it back until you start it yourself.

docker run -p HOST:CONTAINER ... - publish a container port to a host port

docker run -d -P --name web nginx






