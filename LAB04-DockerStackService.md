# Docker Swarm: Single-Node Local Lab Notes

## Purpose

A **single-node Docker Swarm** lets you practice Swarm features on one computer:

- Deploying stacks
- Creating services
- Running replicas
- Scaling services
- Rolling updates
- Self-healing containers/tasks
- Using Swarm overlay networks

Your laptop runs both the **manager** and **worker** roles. This is valid for local learning, but it is not a highly available production cluster. [web:143]

---

## Initialize Swarm

Run this once in PowerShell:

```powershell
docker swarm init
```

There are no flags.

This initializes a new Swarm on the current Docker Engine. The computer becomes the Swarm manager node. [web:138][web:143]

If Docker says that the node is already part of a Swarm, do not run this again.

---

## Confirm Node Status

```powershell
docker node ls
```

There are no flags.

Expected output is similar to:

```text
ID                            HOSTNAME       STATUS    AVAILABILITY   MANAGER STATUS
abc123... *                   docker-desktop Ready     Active         Leader
```

Important fields:

- `STATUS: Ready`: Docker can schedule services on the node.
- `AVAILABILITY: Active`: the node can receive tasks.
- `MANAGER STATUS: Leader`: this node is the active manager for the Swarm.

`docker node ls` must be run from a manager node. [web:139][web:143]

---

## Deploy a Stack

From the folder containing your `stack.yml` file, run:

```powershell
docker stack deploy -c stack.yml wordpress
```

Flag explanation:

- `-c stack.yml`: short for `--compose-file stack.yml`; tells Docker which YAML file defines the stack.
- `wordpress`: the stack name.

The equivalent long form is:

```powershell
docker stack deploy --compose-file stack.yml wordpress
```

Docker uses the stack name as a prefix for resources. For example:

```text
wordpress_mydatabase
wordpress_mywordpress
```

A stack deployment must be run on a Swarm manager node. [web:138][web:139]

---

## Replicas

In `stack.yml`, 

```yaml
services:
  mydatabase:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    volumes:
      - mydata:/var/lib/mysql
    networks:
      - mynet
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-psomewordpress"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  mywordpress:
    image: wordpress:latest
    ports:
      - target: 80
        published: 8080
        protocol: tcp
        mode: ingress
    environment:
      WORDPRESS_DB_HOST: mydatabase:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    networks:
      - mynet
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      restart_policy:
        condition: on-failure

volumes:
  mydata:

networks:
  mynet:
    driver: overlay
```

means:

> “Keep three copies of this service running.”

With only one Swarm node, all three tasks run on the same computer.

Example:

```text
WordPress service
├── Task/container 1
├── Task/container 2
└── Task/container 3
```

If one WordPress task stops, Swarm attempts to create a replacement so the desired count returns to `3/3`.

---

## Check the Stack

List all deployed stacks:

```powershell
docker stack ls
```

There are no flags.

List services in the `wordpress` stack:

```powershell
docker stack services wordpress
```

There are no flags.

Example output:

```text
NAME                    MODE         REPLICAS
wordpress_mydatabase    replicated   1/1
wordpress_mywordpress   replicated   3/3
```

- `1/1`: one running task out of one desired task.
- `3/3`: three running tasks out of three desired tasks.
- `0/1`: the service expects one task, but none are currently running; inspect logs and task status.

List tasks and their status:

```powershell
docker stack ps wordpress
```

There are no flags.

Show full task error messages without shortening them:

```powershell
docker stack ps wordpress --no-trunc
```

Flag explanation:

- `--no-trunc`: do not shorten long output such as task error messages.

---

## Check Service Logs

View database logs:

```powershell
docker service logs wordpress_mydatabase
```

There are no flags.

Follow logs continuously:

```powershell
docker service logs -f wordpress_mydatabase
```

Flag explanation:

- `-f`: follow; continue printing new log entries until you press `Ctrl + C`.

View WordPress logs:

```powershell
docker service logs -f wordpress_mywordpress
```

Flag explanation:

- `-f`: follow logs continuously.

Check why a service task failed:

```powershell
docker service ps wordpress_mydatabase --no-trunc
```

Flag explanation:

- `--no-trunc`: prints the complete error message rather than abbreviating it.

---

## Open the Application

If your stack publishes WordPress like this:

```yaml
ports:
  - target: 80
    published: 8080
    protocol: tcp
    mode: ingress
```

open this URL:

```text
http://localhost:8080
```

- `target: 80`: WordPress listens on port 80 inside the container.
- `published: 8080`: your laptop accepts browser traffic on port 8080.
- `protocol: tcp`: use TCP networking.
- `mode: ingress`: enables Swarm’s routing mesh.

---

## Test Self-Healing

List running local containers:

```powershell
docker container ps
```

There are no flags.

Find a WordPress container, then stop it:

```powershell
docker container stop CONTAINER_ID
```

There are no flags.

Replace `CONTAINER_ID` with the WordPress container ID.

Then check the stack tasks:

```powershell
docker stack ps wordpress
```

There are no flags.

Swarm should create a replacement WordPress task because the desired replica count is three.

---

## Scale WordPress

Scale from three WordPress replicas to five:

```powershell
docker service scale wordpress_mywordpress=5
```

There are no flags.

Verify:

```powershell
docker stack services wordpress
```

There are no flags.

Scale back to three:

```powershell
docker service scale wordpress_mywordpress=3
```

There are no flags.

---

## Remove the Stack

Stop and remove the stack services and network:

```powershell
docker stack rm wordpress
```

There are no flags.

Verify removal:

```powershell
docker stack ls
docker service ls
```

There are no flags.

The named database volume usually remains after removing the stack.

List volumes:

```powershell
docker volume ls
```

There are no flags.

Delete the WordPress database volume permanently:

```powershell
docker volume rm wordpress_mydata
```

There are no flags.

> Warning: Removing this volume permanently deletes the MySQL database files for this lab.

---

## Reset Local Swarm

Only for a disposable local practice environment:

```powershell
docker swarm leave --force
```

Flag explanation:

- `--force`: allows the manager node to leave the Swarm even if it is the only manager.

This removes your local Swarm membership and state. Initialize it again later with:

```powershell
docker swarm init
```

There are no flags.

---

## Single-Node Limitations

A single-node Swarm cannot fully test:

- Scheduling containers across separate machines
- A worker node failing
- Manager quorum or manager failover
- Multi-host overlay-network communication
- High availability when your laptop or Docker Desktop stops

For those features, use multiple Docker hosts, such as separate virtual machines, cloud VMs, physical computers, or Play with Docker instances. A stack deployment is managed by Swarm managers, while worker nodes run scheduled tasks. [web:138][web:139]
