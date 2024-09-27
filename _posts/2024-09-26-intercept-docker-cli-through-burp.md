## Intercepting Docker cli UNIX socket requests through Burp

Docker is now a household name in container technology, and for good reason. The Docker CLI makes managing containers and everything related to them incredibly simple and efficient, making it a go-to tool in my workflow.

Docker CLI uses REST API to connect to Docker Daemon (dockerd), which by default listens for requests on the UNIX socket, ideally located at `/var/run/docker.sock`, to only allow local connections by the root user. We do have another option to expose a TCP port and proxy curl command through it.

We need to have a bridge between the Unix socket and a local TCP port, which will act as a gate to the unix socket.

One of the best tools to create this relay will be ``socat``. We will use socat to create a bridge between a random TCP port and the unix socket. We will then query the Docker Daemon using curl and make sure to proxy the tarffic through the socket used by your local Burp Suite installation.

First we need to install socat on our system. It is readily available on brew and can be easily installed using the command - `` brew install socat``