# Deploying OpenWebUI with Docker

This document provides instructions for deploying OpenWebUI using Docker. You can use either the `docker run` command or Docker Compose.

## Prerequisites

Ensure that Docker is installed and running on your system. You can find installation instructions for your operating system on the [official Docker website](https://docs.docker.com/get-docker/).

## Using `docker run`

This command will download the latest OpenWebUI image and run it in a detached container.

```bash
docker run -d -p <host_port>:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

**Explanation of the command:**

*   `-d`: Runs the container in detached mode (in the background).
*   `-p <host_port>:8080`: Maps port `<host_port>` on your host machine to port 8080 in the container.
    *   You can change `<host_port>` to any available port on your system (e.g., `80`, `8080`, `3000`).
    *   **Important:** If you change `<host_port>` from the default `3000`, you will also need to update the `Valid Redirect URIs` in your Keycloak client configuration and the OpenWebUI public URL (e.g., `OIDC_ISSUER_URL` or similar environment variables if you are configuring OpenWebUI directly) to reflect this new port.
*   `--add-host=host.docker.internal:host-gateway`: Adds a host entry for `host.docker.internal`, which allows the OpenWebUI container to connect to services running on your host machine (like Ollama). This is crucial if Ollama is running directly on your Docker host and not in a container.
*   `-v open-webui:/app/backend/data`: Creates and mounts a Docker volume named `open-webui` to `/app/backend/data` inside the container. This volume is used to persist OpenWebUI's data, such as user information, chat history, and settings, even if the container is removed or updated.
*   `--name open-webui`: Assigns a name to the container for easy reference (e.g., when stopping or removing it).
*   `--restart always`: Configures the container to automatically restart if it stops or if the Docker daemon restarts. This helps ensure OpenWebUI remains available.
*   `ghcr.io/open-webui/open-webui:main`: Specifies the Docker image to use. This pulls the `main` tag (often the latest stable version) from the GitHub Container Registry.

## Using `docker-compose.yml`

Docker Compose allows you to define and manage multi-container Docker applications. Create a file named `docker-compose.yml` with the following content:

```yaml
version: '3.8'

services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "<host_port>:8080" # Change <host_port> if needed (e.g., "3000:8080", "80:8080")
    volumes:
      - open-webui:/app/backend/data
    # This allows OpenWebUI to connect to Ollama or other services running on the host machine.
    # It maps host.docker.internal to the host's IP address within the container.
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: always

volumes:
  open-webui:
    # This defines the named volume 'open-webui' for persistent data storage.
    # Docker will manage this volume.
```

**To use Docker Compose:**

1.  Save the content above into a file named `docker-compose.yml` in a directory of your choice.
2.  Open a terminal, navigate to that directory.
3.  Run the command: `docker-compose up -d`

**Explanation of the `docker-compose.yml` file:**

*   `version: '3.8'`: Specifies the version of the Docker Compose file format.
*   `services:`: Defines the different services (containers) that make up your application.
    *   `open-webui:`: Defines the OpenWebUI service.
        *   `image: ghcr.io/open-webui/open-webui:main`: Specifies the Docker image to use.
        *   `container_name: open-webui`: Sets the name of the container.
        *   `ports:`: Maps ports between the host and the container.
            *   `"<host_port>:8080"`: Maps `<host_port>` on the host to port 8080 in the container. Remember to replace `<host_port>` (e.g., `3000`).
            *   **Important:** If you change `<host_port>` from the default `3000`, you will also need to update the `Valid Redirect URIs` in your Keycloak client configuration and the OpenWebUI public URL.
        *   `volumes:`: Mounts volumes for persistent data.
            *   `open-webui:/app/backend/data`: Mounts the named volume `open-webui` to `/app/backend/data` in the container.
        *   `extra_hosts:`: Adds custom host mappings.
            *   `"host.docker.internal:host-gateway"`: Allows connection to services on the host machine.
        *   `restart: always`: Ensures the container restarts automatically.
*   `volumes:`: Defines named volumes.
    *   `open-webui:`: Declares the `open-webui` volume, which Docker will manage for data persistence.

By default, OpenWebUI will be accessible at `http://localhost:<host_port>` after running either of these commands. Remember to replace `<host_port>` with the actual port number you've chosen (e.g., `http://localhost:3000`).
