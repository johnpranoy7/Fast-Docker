# Docker Local Registry Lab (Windows + Docker Desktop)

## Goal

This lab creates a free private Docker registry on your laptop, then shows how to:

- Run the registry
- Persist registry data on the Windows host
- Tag an image for the local registry
- Push an image to it
- List stored repositories
- Pull the image from it
- Stop and remove the registry safely

A local registry is available at:

```text
http://localhost:5000
```

The Docker Registry image uses the `registry` image name. The `:2` suffix is the Registry v2 image tag, which is commonly used for a local registry. [web:154][web:171]

---

## Prerequisites

- Docker Desktop is running.
- Docker Desktop is in **Linux containers** mode.
- PowerShell is open.

Verify Docker is available:

```powershell
docker version
```

There are no flags.

---

## 1. Check Existing Containers

List all containers before creating the registry:

```powershell
docker container ls -a
```

Flag explanation:

- `-a`: **all**; shows both running and stopped containers.

If a previous lab container named `localregistry` exists, remove it:

```powershell
docker container rm -f localregistry
```

Flag explanation:

- `-f`: **force**; stops a running container if necessary, then removes it.

> Only run this if you want to delete and recreate the old registry container.

---

## 2. Create Persistent Storage

Create a Windows folder where the registry stores pushed image layers and metadata:

```powershell
New-Item -ItemType Directory -Force C:\docker_registry
```

Flag explanation:

- `-ItemType Directory`: create a folder.
- `-Force`: do not fail if `C:\docker_registry` already exists.

---

## 3. Download the Registry Image

```powershell
docker image pull registry:2
```

There are no flags.

- `registry`: official Docker Registry image.
- `:2`: Registry v2 image tag.

If Docker prints:

```text
Unable to find image 'registry:2' locally
```

that is normal on the first run. Docker then downloads it from Docker Hub.

Verify the image exists locally:

```powershell
docker image ls registry
```

There are no flags.

---

## 4. Start the Local Registry

Run exactly one registry container:

```powershell
docker run -d -p 5000:5000 --restart=always --name localregistry -v "C:\docker_registry:/var/lib/registry" -e REGISTRY_STORAGE_DELETE_ENABLED=true registry:2
```

Flag explanations:

- `-d`: **detached mode**; runs the container in the background.
- `-p 5000:5000`: maps port `5000` on the laptop to port `5000` inside the Registry container.
- `--restart=always`: automatically restarts the Registry if it exits or Docker Desktop restarts.
- `--name localregistry`: assigns the container the name `localregistry`.
- `-v "C:\docker_registry:/var/lib/registry"`: bind-mounts the Windows folder into the registry storage directory inside the container.
- `-e REGISTRY_STORAGE_DELETE_ENABLED=true`: sets an environment variable that enables image deletion through the Registry HTTP API.
- `registry:2`: runs the Registry v2 image.

> Do not also run a second Registry container on port `5000`. Only one container can publish the same host port at a time.

Check that it is running:

```powershell
docker container ps
```

There are no flags.

Expected output includes:

```text
localregistry
0.0.0.0:5000->5000/tcp
```

---

## 5. View Registry Logs

Show Registry logs:

```powershell
docker container logs localregistry
```

There are no flags.

Follow logs continuously:

```powershell
docker container logs -f localregistry
```

Flag explanation:

- `-f`: **follow**; continuously display new log lines until `Ctrl + C` is pressed.

---

## 6. Verify the Empty Registry

Use the Registry HTTP API to list stored repositories:

```powershell
Invoke-RestMethod http://localhost:5000/v2/_catalog
```

There are no flags.

Initially, expected output resembles:

```json
{
  "repositories": []
}
```

---

## 7. Pull an Image to Your Laptop

Pull Alpine from Docker Hub:

```powershell
docker image pull alpine:latest
```

There are no flags.

- `alpine`: image repository name.
- `:latest`: image tag.

Verify it exists locally:

```powershell
docker image ls alpine
```

There are no flags.

---

## 8. Tag Image for Local Registry

Tag the existing Alpine image with the local registry address:

```powershell
docker image tag alpine:latest localhost:5000/alpine:latest
```

There are no flags.

The new name tells Docker where the image should be pushed:

```text
localhost:5000/alpine:latest
^^^^^^^^^^^^^^
local Docker Registry
```

Verify both Alpine tags:

```powershell
docker image ls alpine
docker image ls localhost:5000/alpine
```

There are no flags.

---

## 9. Push Image to Local Registry

```powershell
docker image push localhost:5000/alpine:latest
```

There are no flags.

Docker uploads the image layers to the Registry container, which stores them under:

```text
C:\docker_registry
```

List repository names in the registry:

```powershell
Invoke-RestMethod http://localhost:5000/v2/_catalog
```

Expected result:

```json
{
  "repositories": [
    "alpine"
  ]
}
```

List the image tags stored for Alpine:

```powershell
Invoke-RestMethod http://localhost:5000/v2/alpine/tags/list
```

There are no flags.

Expected result:

```json
{
  "name": "alpine",
  "tags": [
    "latest"
  ]
}
```

---

## 10. Test Pulling from Registry

Remove the local Registry-tagged image:

```powershell
docker image rm localhost:5000/alpine:latest
```

There are no flags.

Pull it again, now from your local Registry:

```powershell
docker image pull localhost:5000/alpine:latest
```

There are no flags.

Docker should report that the image is pulled from `localhost:5000`, not Docker Hub.

Run it to confirm it works:

```powershell
docker run --rm localhost:5000/alpine:latest echo "Hello from my local registry"
```

Flag explanation:

- `--rm`: automatically remove the temporary container after its command finishes.
- `localhost:5000/alpine:latest`: image pulled from the local registry.
- `echo "Hello from my local registry"`: command run inside the Alpine container.

---

## 11. Push a Second Image

Pull BusyBox:

```powershell
docker image pull busybox:latest
```

There are no flags.

Tag it:

```powershell
docker image tag busybox:latest localhost:5000/busybox:latest
```

There are no flags.

Push it:

```powershell
docker image push localhost:5000/busybox:latest
```

There are no flags.

View all repositories:

```powershell
Invoke-RestMethod http://localhost:5000/v2/_catalog
```

Expected result:

```json
{
  "repositories": [
    "alpine",
    "busybox"
  ]
}
```

---

## 12. Stop and Restart Registry

Stop the Registry container:

```powershell
docker container stop localregistry
```

There are no flags.

Start it again:

```powershell
docker container start localregistry
```

There are no flags.

Your pushed images remain available because the data is stored in:

```text
C:\docker_registry
```

---

## 13. Remove Registry Container

Remove the Registry container but keep the stored image data:

```powershell
docker container rm -f localregistry
```

Flag explanation:

- `-f`: **force**; stop the container if it is running, then remove it.

The data remains because it was bind-mounted to:

```text
C:\docker_registry
```

Recreate the Registry using the same command:

```powershell
docker run -d -p 5000:5000 --restart=always --name localregistry -v "C:\docker_registry:/var/lib/registry" -e REGISTRY_STORAGE_DELETE_ENABLED=true registry:2
```

Then confirm repositories still exist:

```powershell
Invoke-RestMethod http://localhost:5000/v2/_catalog
```

There are no flags.

---

## 14. Full Cleanup

Stop and remove the Registry container:

```powershell
docker container rm -f localregistry
```

Flag explanation:

- `-f`: force-stop and remove the container.

Remove local tags and images if desired:

```powershell
docker image rm localhost:5000/alpine:latest
docker image rm localhost:5000/busybox:latest
```

There are no flags.

To permanently erase every pushed image from the local Registry, delete its storage directory:

```powershell
Remove-Item -Recurse -Force C:\docker_registry
```

Flag explanations:

- `-Recurse`: delete the directory and everything inside it.
- `-Force`: delete without confirmation and include hidden items.

> Warning: This permanently deletes all Registry image data.

---

## Important Notes

- Use `localhost:5000` consistently for this lab.
- Do not use both `localhost:5000/alpine` and `127.0.0.1:5000/alpine` interchangeably; Docker treats them as different image names.
- Do not manually delete folders inside `/var/lib/registry` while the registry is active.
- A registry without TLS or authentication is appropriate only for a local development machine, not a public or production server.
- The Windows-container portion of the older lab is separate and not necessary for learning Linux Docker images and local Registry fundamentals.
