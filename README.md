Traefik Proxy with HTTPS
This article lives in:

Medium
GitHub
DockerSwarm.rocks
Intro
So, you have a Docker Swarm mode cluster set up as described in DockerSwarm.rocks.

Now you can add a main, distributed, Traefik load balancer/proxy to:

Handle connections.
Expose specific services and applications based on their domain names.
Handle multiple domains (if you need to). Similar to "virtual hosts".
Handle HTTPS.
Acquire (generate) HTTPS certificates automatically (including renewals) with Let's Encrypt.
Add HTTP Basic Auth for any service that you need to protect and doesn't have its own security, etc.
Get all its configurations automatically from Docker labels set in your stacks (you don't need to update configuration files).
This article/guide covers setting up Traefik in a distributed system, including distributed HTTPS certificates.

These ideas, techniques, and tools would also apply to other cluster orchestrators, like Kubernetes or Mesos, to add a main load balancer with HTTPS support, certificate generation, etc. But this article is focused on Docker Swarm mode.

It's an alternative/continuation to a previous article Docker Swarm Mode and Traefik for an HTTPS cluster that covered Traefik in a Docker Swarm mode cluster but running on a single node.

Background
Docker Swarm mode with a main Traefik load balancer/proxy is the base cluster architecture that I'm using with my current team for most of the applications and projects.

It's also used by several other friends and teams.

Overview
This guide will show you how to set up Traefik as a load balancer/proxy and Consul to store configurations and HTTPS certificates.

Redundancy
As Traefik and Consul are both distributed, it doesn't matter if one of your nodes/machines goes down, the distributed Traefik will be able to handle it from another of the nodes running, preserving HTTPS certificates, etc.

So, you have "redundancy" in the load balancer in your cluster.

You can easily have redundancy in your application using Docker Swarm.

And if you set your domain name DNS records correctly, adding the IP addresses of several of the machines in your cluster, you would then have round-robin DNS load balancing for your application too.

Full application redundancy.

Note

But if you have a single node, it will also work. And you can grow later if needed.

User Interface
The guide includes how to expose the internal Traefik web UI through the same Traefik load balancer, using a secure HTTPS certificate and HTTP Basic Auth.



How it works
The idea is to have a main load balancer/proxy that covers all the Docker Swarm cluster and handles HTTPS certificates and requests for each domain.

But doing it in a way that allows you to have other Traefik services inside each stack without interfering with each other, to redirect based on path in the same stack (e.g. one container handles / for a web frontend and another handles /api for an API under the same domain), or to redirect from HTTP to HTTPS selectively.

About Consul
Consul is a distributed configuration key/value store, it will:

Store the Traefik configurations in a distributed manner.
Make sure configurations are synchronized among Consul services across the cluster.
Store HTTPS certificates for Traefik.
Preparation
Connect via SSH to a manager node in your cluster (you might have only one node) that will have the Traefik service.
Create a network that will be shared with Traefik and the containers that should be accessible from the outside, with:
Copied to clipboard
docker network create --driver=overlay traefik-public
Create an environment variable with your email, to be used for the generation of Let's Encrypt certificates:

export EMAIL=admin@example.com
Create an environment variable with the domain you want to use for the Traefik UI (user interface) and the Consul UI of the host, e.g.:

export DOMAIN=sys.example.com
You will access the Traefik UI at traefik.<your domain>, e.g. traefik.sys.example.com and the Consul UI at consul.<your domain>, e.g. consul.sys.example.com.

So, make sure that your DNS records point traefik.<your domain> and consul.<your domain> to one of the IPs of the cluster.

If you have several nodes (several IP addresses), you might want to create the DNS records for multiple of those IP addresses.

That way, you would have redundancy even at the DNS level.

Create an environment variable with a username (you will use it for the HTTP Basic Auth for Traefik and Consul UIs), for example:

export USERNAME=admin
Create an environment variable with the password, e.g.:

export PASSWORD=changethis
Use openssl to generate the "hashed" version of the password and store it in an environment variable:

export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)
You can check the contents with:

echo $HASHED_PASSWORD
It will look like:


$apr1$89eqM5Ro$CxaFELthUKV21DpI3UTQO.
Create an environment variable with the number of replicas for the Consul service (if you don't set it, by default it will be 3):

export CONSUL_REPLICAS=3
If you have a single node, you can set CONSUL_REPLICAS to 0, that way you will only have the Consul "leader", you don't need the replicas if you don't have other nodes yet:


export CONSUL_REPLICAS=0
Note: The Consul documentation says:

It is expected that there be between three to five servers.

So, you probably want to set CONSUL_REPLICAS=3 or CONSUL_REPLICAS=5, but not more.

Create an environment variable with the number of replicas for the Traefik service (if you don't set it, by default it will be 3):

export TRAEFIK_REPLICAS=3
...if you want to have one replica per node in your cluster, you can set it like this:


export TRAEFIK_REPLICAS=$(docker node ls -q | wc -l)
...if you have a single node, you can set TRAEFIK_REPLICAS to 1:


export TRAEFIK_REPLICAS=1
Create the Docker Compose file
Download the file traefik.yml:

curl -L dockerswarm.rocks/traefik.yml -o traefik.yml
...or create it manually, for example, using nano:

nano traefik.yml
And copy the contents inside:

version: '3.3'

services:
  consul-leader:
    image: consul
    command: agent -server -client=0.0.0.0 -bootstrap -ui
    volumes:
      - consul-data-leader:/consul/data
    environment:
      - CONSUL_BIND_INTERFACE=eth0
      - 'CONSUL_LOCAL_CONFIG={"leave_on_terminate": true}'
    networks:
      - default
      - traefik-public
    deploy:
      labels:
        - traefik.frontend.rule=Host:consul.${DOMAIN?Variable DOMAIN not set}
        - traefik.enable=true
        - traefik.port=8500
        - traefik.tags=${TRAEFIK_PUBLIC_TAG:-traefik-public}
        - traefik.docker.network=traefik-public
        # Traefik service that listens to HTTP
        - traefik.redirectorservice.frontend.entryPoints=http
        - traefik.redirectorservice.frontend.redirect.entryPoint=https
        # Traefik service that listens to HTTPS
        - traefik.webservice.frontend.entryPoints=https
        - traefik.frontend.auth.basic.users=${USERNAME?Variable USERNAME not set}:${HASHED_PASSWORD?Variable HASHED_PASSWORD not set}
  consul-replica:
    image: consul
    command: agent -server -client=0.0.0.0 -retry-join="consul-leader"
    volumes:
      - consul-data-replica:/consul/data
    environment:
      - CONSUL_BIND_INTERFACE=eth0
      - 'CONSUL_LOCAL_CONFIG={"leave_on_terminate": true}'
    networks:
      - default
      - traefik-public
    deploy:
      replicas: ${CONSUL_REPLICAS:-3}
      placement:
        preferences:
          - spread: node.id
  traefik:
    image: traefik:v1.7
    ports:
      - 80:80
      - 443:443
    deploy:
      replicas: ${TRAEFIK_REPLICAS:-3}
      placement:
        constraints:
          - node.role == manager
        preferences:
          - spread: node.id
      labels:
        - traefik.frontend.rule=Host:traefik.${DOMAIN?Variable DOMAIN not set}
        - traefik.enable=true
        - traefik.port=8080
        - traefik.tags=traefik-public
        - traefik.docker.network=traefik-public
        # Traefik service that listens to HTTP
        - traefik.redirectorservice.frontend.entryPoints=http
        - traefik.redirectorservice.frontend.redirect.entryPoint=https
        # Traefik service that listens to HTTPS
        - traefik.webservice.frontend.entryPoints=https
        - traefik.frontend.auth.basic.users=${USERNAME?Variable USERNAME not set}:${HASHED_PASSWORD?Variable HASHED_PASSWORD not set}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: >
      --docker
      --docker.swarmmode
      --docker.watch
      --docker.exposedbydefault=false
      --constraints=tag==traefik-public
      --entrypoints='Name:http Address::80'
      --entrypoints='Name:https Address::443 TLS'
      --consul
      --consul.endpoint="consul-leader:8500"
      --acme
      --acme.email=${EMAIL?Variable EMAIL not set}
      --acme.storage="traefik/acme/account"
      --acme.entryPoint=https
      --acme.httpChallenge.entryPoint=http
      --acme.onhostrule=true
      --acme.acmelogging=true
      --logLevel=INFO
      --accessLog
      --api
    networks:
      - default
      - traefik-public
    depends_on:
      - consul-leader

volumes:
  consul-data-leader:
  consul-data-replica:

networks:
  traefik-public:
    external: true
Info

This is just a standard Docker Compose file.

It's common to name the file docker-compose.yml or something like docker-compose.traefik.yml.

Here it's named just traefik.yml for brevity.

Deploy it
Deploy the stack with:


docker stack deploy -c traefik.yml traefik-consul
It will use the environment variables you created above.

Check it
Check if the stack was deployed with:

docker stack ps traefik-consul
It will output something like:


ID             NAME                              IMAGE           NODE                DESIRED STATE   CURRENT STATE          ERROR   PORT
xvyasdfh56hg   traefik-consul_traefik.1          traefik:v1.7    dog.example.com     Running         Running 1 minute ago
j3ahasdfe0mr   traefik-consul_consul-replica.1   consul:latest   cat.example.com     Running         Running 1 minute ago
bfdasdfasr92   traefik-consul_consul-leader.1    consul:latest   cat.example.com     Running         Running 1 minute ago
ofvasdfqtsi6   traefik-consul_traefik.2          traefik:v1.7    snake.example.com   Running         Running 1 minute ago
tybasdfqdutt   traefik-consul_consul-replica.2   consul:latest   dog.example.com     Running         Running 1 minute ago
3ejasdfq2l3g   traefik-consul_traefik.3          traefik:v1.7    cat.example.com     Running         Running 1 minute ago
w0oasdfqsv33   traefik-consul_consul-replica.3   consul:latest   snake.example.com   Running         Running 1 minute ago
You can check the Traefik logs with:

docker service logs traefik-consul_traefik
Check the user interfaces
After some seconds/minutes, Traefik will acquire the HTTPS certificates for the web user interfaces.

You will be able to securely access the web UI at https://traefik.<your domain> using the created username and password.

And the same way, to access the Consul web user interface at https://consul.<your domain>.

Updating
Let's say you add a couple of new nodes to your cluster, and you want to increment the number of Consul replicas or Traefik replicas.

You can just set the environment variables again, and re-deploy. Docker Swarm mode will take care of making sure the state of the system is consistent.

Getting the client IP
If you need to read the client IP in your applications/stacks using the X-Forwarded-For or X-Real-IP headers provided by Traefik, you need to make Traefik listen directly, not through Docker Swarm mode, even while being deployed with Docker Swarm mode.

For that, you need to publish the ports using "host" mode.

So, the Docker Compose lines:


    ports:
      - 80:80
      - 443:443
need to be:


    ports:
      - target: 80
        published: 80
        mode: host
      - target: 443
        published: 443
        mode: host
You can use all the same instructions above, downloading the host-mode file:


curl -L dockerswarm.rocks/traefik-host.yml -o traefik-host.yml
Or alternatively, copying it directly:

Copied to clipboard
version: '3.3'

services:
  consul-leader:
    image: consul
    command: agent -server -client=0.0.0.0 -bootstrap -ui
    volumes:
      - consul-data-leader:/consul/data
    environment:
      - CONSUL_BIND_INTERFACE=eth0
      - 'CONSUL_LOCAL_CONFIG={"leave_on_terminate": true}'
    networks:
      - default
      - traefik-public
    deploy:
      labels:
        - traefik.frontend.rule=Host:consul.${DOMAIN?Variable DOMAIN not set}
        - traefik.enable=true
        - traefik.port=8500
        - traefik.tags=${TRAEFIK_PUBLIC_TAG:-traefik-public}
        - traefik.docker.network=traefik-public
        # Traefik service that listens to HTTP
        - traefik.redirectorservice.frontend.entryPoints=http
        - traefik.redirectorservice.frontend.redirect.entryPoint=https
        # Traefik service that listens to HTTPS
        - traefik.webservice.frontend.entryPoints=https
        - traefik.frontend.auth.basic.users=${USERNAME?Variable USERNAME not set}:${HASHED_PASSWORD?Variable HASHED_PASSWORD not set}
  consul-replica:
    image: consul
    command: agent -server -client=0.0.0.0 -retry-join="consul-leader"
    volumes:
      - consul-data-replica:/consul/data
    environment:
      - CONSUL_BIND_INTERFACE=eth0
      - 'CONSUL_LOCAL_CONFIG={"leave_on_terminate": true}'
    networks:
      - default
      - traefik-public
    deploy:
      replicas: ${CONSUL_REPLICAS:-3}
      placement:
        preferences:
          - spread: node.id
  traefik:
    image: traefik:v1.7
    ports:
      - target: 80
        published: 80
        mode: host
      - target: 443
        published: 443
        mode: host
    deploy:
      replicas: ${TRAEFIK_REPLICAS:-3}
      placement:
        constraints:
          - node.role == manager
        preferences:
          - spread: node.id
      labels:
        - traefik.frontend.rule=Host:traefik.${DOMAIN?Variable DOMAIN not set}
        - traefik.enable=true
        - traefik.port=8080
        - traefik.tags=traefik-public
        - traefik.docker.network=traefik-public
        # Traefik service that listens to HTTP
        - traefik.redirectorservice.frontend.entryPoints=http
        - traefik.redirectorservice.frontend.redirect.entryPoint=https
        # Traefik service that listens to HTTPS
        - traefik.webservice.frontend.entryPoints=https
        - traefik.frontend.auth.basic.users=${USERNAME?Variable USERNAME not set}:${HASHED_PASSWORD?Variable HASHED_PASSWORD not set}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: >
      --docker
      --docker.swarmmode
      --docker.watch
      --docker.exposedbydefault=false
      --constraints=tag==traefik-public
      --entrypoints='Name:http Address::80'
      --entrypoints='Name:https Address::443 TLS'
      --consul
      --consul.endpoint="consul-leader:8500"
      --acme
      --acme.email=${EMAIL?Variable EMAIL not set}
      --acme.storage="traefik/acme/account"
      --acme.entryPoint=https
      --acme.httpChallenge.entryPoint=http
      --acme.onhostrule=true
      --acme.acmelogging=true
      --logLevel=INFO
      --accessLog
      --api
    networks:
      - default
      - traefik-public
    depends_on:
      - consul-leader

volumes:
  consul-data-leader:
  consul-data-replica:

networks:
  traefik-public:
    external: true
And then deploying with:

Copied to clipboard
docker stack deploy -c traefik-host.yml traefik-consul
What's next
The next thing would be to deploy a stack (a complete web application, with backend, frontend, database, etc) using this Docker Swarm mode cluster.

It's actually very simple, as you can use Docker Compose for local development and then use the same files for deployment in the Docker Swarm mode cluster.

If you want to try it right now, you can check this very simple project generator with a minimal Flask backend and Vue.js frontend.

It has everything set up to be deployed in a Docker Swarm mode cluster with Traefik as described in this article.

Technical Details
If you want to see the technical details of what each part of the Docker Compose traefik.yml file do, check the chapter "Traefik Proxy with HTTPS - Technical Details".
