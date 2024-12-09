# Adobe Dispatcher Container Image

Adobe includes a prebuilt container image in their SDK archives. However, this image 
is not hosted on any publicly accessible registry. To address this, this repository 
offers a method to push the prebuilt image to a private container registry, making it 
accessible for private development projects.

# Push Adobe Image To Private Registry

The snippets below demonstrate how to load images locally, apply a custom tag, create 
a manifest for multi-architecture support, and push the image to a registry. You must 
first log in to your container registry. In the examples below, we will demonstrate 
using the GitHub Container Registry.

```shell
export TOKEN=***
export REGISTRY=ghcr.io
export USER=<your github username here>

echo ${TOKEN} | podman login ${REGISTRY} -u ${USER} --password-stdin
```

Next we load the containers from the SDK archive, tag it with their respective 
architecture and push it to the GH registry.

```shell
cd <sdk-zip>/dispatcher-sdk-*/lib

# amd64
podman load < dispatcher-publish-amd64.tar.gz
podman tag docker.io/adobe/aem-cs/dispatcher-publish:2.0.232 \
           ${REGISTRY}/${USER}/dispatcher:2.0.232-amd64
podman push ${REGISTRY}/${USER}/dispatcher:2.0.232-amd64

#arm64
podman load < dispatcher-publish-arm64.tar.gz
podman tag docker.io/adobe/aem-cs/dispatcher-publish:2.0.232 \
           ${REGISTRY}/${USER}/dispatcher:2.0.232-arm64
podman push ${REGISTRY}/${USER}/dispatcher:2.0.232-arm64
```

After both architecture-specific images have been pushed to the registry, a manifest 
can be created to automatically select the appropriate architecture when the image is 
pulled in a development environment. 

```shell
podman manifest create ${REGISTRY}/${USER}/dispatcher:2.0.232 \
                       ${REGISTRY}/${USER}/dispatcher:2.0.232-amd64 \
                       ${REGISTRY}/${USER}/dispatcher:2.0.232-arm64

podman manifest push ${REGISTRY}/${USER}/dispatcher:2.0.232 \
                     ${REGISTRY}/${USER}/dispatcher:2.0.232
podman manifest push ${REGISTRY}/${USER}/dispatcher:2.0.232 \
                     ${REGISTRY}/${USER}/dispatcher:2024.11
podman manifest push ${REGISTRY}/${USER}/dispatcher:2.0.232 \
                     ${REGISTRY}/${USER}/dispatcher:latest
```

From this point onward, the manifest in the private registry can be referenced directly 
and seamlessly integrated into a development setup (e.g., Podman or Docker Compose). 
Additionally, it automatically resolves to the correct architecture (e.g., ARM64 for 
Apple Silicon).

> Using architecture-native images can significantly enhance performance, making it 
> highly recommended.

# How To Use

You can pull the images from your container registry. We differentiate between
a `stable` version, which represents the production installation, and the `latest`
version, which aligns with the Cloud Manager development environment (and potentially
RDE). This approach allows you to test your code with the upcoming SDK version before
upgrading your production environment. In addition to these two tags, we also generate 
tags that correspond to the SDK version (e.g., `2024.11`) and the specific version of 
the Adobe Dispatcher image (e.g., `2.0.232`).

```shell
podman pull ${REGISTRY}/${USER}/dispatcher:stable
podman pull ${REGISTRY}/${USER}/dispatcher:2024.10

podman pull ${REGISTRY}/${USER}/dispatcher:2.0.232

podman pull ${REGISTRY}/${USER}/dispatcher:latest
podman pull ${REGISTRY}/${USER}/dispatcher:2024.11
```

Once you have successfully pulled the container image from the registry, you can proceed
to start the containers. Be sure to adjust the arguments to suit your specific requirements.

```shell
podman run -d \
  --name dispatcher \
  -e TZ=Europe/Zurich \
  -e AEM_HOST="publish" \
  -e AEM_IP="*" \
  -e AEM_PORT="4500" \
  -e ALLOW_CACHE_INVALIDATION_GLOBALLY="true" \
  -e DISP_RUN_MODE="dev" \
  -e ENVIRONMENT_TYPE="dev" \
  -e HOT_RELOAD="true" \
  -e REWRITE_LOG_LEVEL="debug" \
  -p 8000:80 \
  -v $(pwd)/dispatcher/logs:/var/log/apache2 \
  -v $(pwd)/dispatcher/cache:/mnt/var/www \
  -v <path/to/archetype/project/dispatcher/src>:/mnt/dev/src:ro \
  ${REGISTRY}/${USER}/dispatcher:latest
```

Once the containers are created you can use the following commands to start/stop
the containers.

```shell
docker start dispatcher
docker stop dispatcher (-t 30)
```
