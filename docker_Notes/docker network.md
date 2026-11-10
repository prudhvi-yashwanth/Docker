bridge, host, and none. The one that matters most is bridge, the default network every container joins unless you say otherwise.

When you run a container with no network options, it joins the default bridge network and gets a private IP address on it. Containers on the bridge can reach each other by IP address, and Docker connects the bridge to the outside so containers can reach the internet and you can publish ports to the host.
docker network ls - list networks
A container can even sit on more than one network at once,

docker network create NAME - create a user-defined network
docker run --network NAME ... - attach a container at run time
docker network connect / disconnect NAME CONTAINER - attach or detach a running container
docker network inspect NAME - show a network's details

Host to container - needs a published port (-p), addressed as localhost:HOSTPORT.
Container to container - needs a shared user-defined network, addressed by container name on the service's real port. No -p required.
