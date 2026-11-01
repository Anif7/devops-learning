# Automated Server Provisioning with Cloud-Init

## Overview

This project demonstrates how to automatically set up an Ubuntu server using **cloud-init**.

Instead of manually installing packages, creating users, configuring services, and deploying an application, we define all the required setup steps in a single `cloud-init.yaml` file.

When a new virtual machine starts, cloud-init reads this file and performs the setup automatically.

The project uses:
- **Multipass** to create an Ubuntu 24.04 virtual machine.
- **Cloud-init** to configure the server.
- **Docker** to run a simple HTTP application.
- **Systemd** to manage the application and start it automatically when the server boots.

The application listens on port `8080` and returns `Hello from Testpress!` when accessed.

## Prerequisites

Before starting, make sure the following tools are installed:

- **Multipass** – To create and manage Ubuntu virtual machines.
- **Ubuntu 24.04** – The operating system used for this exercise.
- **Cloud-init** – Used to configure the virtual machine during its first boot. Multipass provides cloud-init support.

No paid cloud infrastructure is required. Everything runs locally on your machine.

### Verify Multipass Installation

Run the following command to check whether Multipass is installed:

```bash
multipass version
```

Check whether Multipass is working:

```bash
multipass list
```


## Project Structure

The project contains the following files and directories:

```text
.
├── cloud-init.yaml
└── README.md
```

- **cloud-init.yaml** – Contains the server provisioning instructions, including package installation, user creation, application files, Docker configuration, and systemd service setup.
- **README.md** – Explains how to create the virtual machine, reproduce the setup, and verify the application.

The application files and configuration are created automatically inside the VM during provisioning.


## Create the Virtual Machine

Create an Ubuntu 24.04 virtual machine using Multipass.

Run the following command from the directory containing `cloud-init.yaml`:

```bash
multipass launch 24.04 \
  --name testpress-server \
  --cpus 2 \
  --memory 2G \
  --disk 5G \
  --cloud-init cloud-init.yaml
```

This command:

- Creates a new virtual machine using Ubuntu 24.04.
- Sets the VM name to `testpress-server`.
- Allocates 2 CPUs, 2 GB RAM, and 5 GB disk space.
- Passes `cloud-init.yaml` to the VM as its provisioning configuration.

Multipass starts the VM, and cloud-init automatically executes the configuration during the initial boot.

**Note:** The provisioning process may take a few minutes because it needs to install packages, create application files, build the Docker image, and start the application.


## Cloud-Init Configuration

The `cloud-init.yaml` file contains all the instructions required to provision and configure the server automatically.

During the initial boot, cloud-init processes the configuration and performs the following tasks:

1. **Timezone:** Sets the server timezone to UTC.

2. **Package Installation:** Updates the package index and installs Docker, Git, curl, and FFmpeg.

3. **User and Group Setup:** Ensures the Docker group exists, keeps the default user, and creates a non-root user named `testpress`. The new user is added to the Docker group and password-based login is disabled.

4. **Create Application Files:** Creates the application directory and writes the required files:
   - `app.env` – Contains sample application environment variables.
   - `server.py` – Runs a simple HTTP server on port `8080` and provides application and health-check endpoints.
   - `Dockerfile` – Defines how to build the Docker image, including the Python environment, non-root application user, and health check.

   The files use specific ownership and permissions. The `defer` option delays writing selected files until the required users and groups are available.

5. **Configure Systemd Service:** Creates `testpress-app.service` to manage the Docker container. The service starts after Docker and network initialization, automatically starts during system boot, and restarts if the application stops unexpectedly.

6. **Execute Provisioning Commands:** The `runcmd` section runs the final setup commands:
   - Sets ownership of the application directories to `testpress`.
   - Builds the Docker image using the Dockerfile.
   - Reloads systemd to recognize the new service.
   - Enables and starts the application service.

7. **Final Message:** Displays a message indicating that cloud-init provisioning has finished.

All these steps are defined in a single configuration file, allowing a fresh virtual machine to be provisioned without manually installing or configuring anything.


## Verify the Provisioning

After the VM is created, check its status:

```bash
multipass list
```

The `testpress-server` VM should be in the `Running` state.

Check the cloud-init status:

```bash
multipass exec testpress-server -- cloud-init status --long
```

Cloud-init should report:

```text
status: done
```

This indicates that cloud-init has completed its provisioning tasks.

To check whether the required packages are installed:

```bash
multipass exec testpress-server -- docker --version
multipass exec testpress-server -- git --version
multipass exec testpress-server -- curl --version
multipass exec testpress-server -- ffmpeg -version
```

To verify that the `testpress` user belongs to the Docker group:

```bash
multipass exec testpress-server -- id testpress
```

The output should include `docker` in the user's groups.



## Verify the Application

Check whether the application container is running:

```bash
multipass exec testpress-server -- sudo -u testpress docker ps
```

The output should show the `testpress-app` container running and port `8080` mapped to the host.

Get the VM's IP address:

```bash
multipass info testpress-server
```

Use the IP address to access the application from your host machine:

```bash
curl http://<VM-IP>:8080/
```

Expected response:

```text
Hello from Testpress!
```

Verify the health-check endpoint:

```bash
curl http://<VM-IP>:8080/health
```

Expected response:

```text
ok
```

You can also verify that an unknown URL returns a `404 Not Found` response:

```bash
curl -i http://<VM-IP>:8080/unknown
```

This confirms that the application is accessible over HTTP and its endpoints are responding as expected.


## Verify Automatic Startup

Restart the virtual machine:

```bash
multipass restart testpress-server
```

Wait for the VM to start, then check the application container:

```bash
multipass exec testpress-server -- sudo -u testpress docker ps
```

The output should show the `testpress-app` container running.

You can also check the systemd service status:

```bash
multipass exec testpress-server -- sudo systemctl status testpress-app.service
```

The service should show `active (running)`.

Finally, verify that the application is accessible after the restart:

```bash
curl http://<VM-IP>:8080/
```

Expected response:

```text
Hello from Testpress!
```

This confirms that systemd automatically starts the application when the VM boots.


## Verify Idempotency

Cloud-init is designed to run its initialization tasks during the first boot of a virtual machine. Running cloud-init commands again should not unnecessarily duplicate users or application files.

Check whether the `testpress` user already exists:

```bash
multipass exec testpress-server -- id testpress
```

Check whether the application files exist:

```bash
multipass exec testpress-server -- sudo ls -l /opt/testpress/app
```

Check whether the systemd service is enabled:

```bash
multipass exec testpress-server -- sudo systemctl is-enabled testpress-app.service
```

Expected output:

```text
enabled
```

Check the cloud-init status:

```bash
multipass exec testpress-server -- cloud-init status
```

Expected output:

```text
status: done
```

These checks confirm that the user, application files, and service are already present after provisioning.

**Note:** Cloud-init normally runs its first-boot configuration only once. Rebooting the VM does not execute all provisioning steps again. These checks verify that the existing setup remains available; they do not prove that running the entire provisioning process again is safe.


## Troubleshooting

If something does not work as expected, use the following commands to investigate the issue.

**1. Check cloud-init logs**

Display the complete cloud-init output log:

```bash
multipass exec testpress-server -- sudo cat /var/log/cloud-init-output.log
```

Display only the last 50 lines:

```bash
multipass exec testpress-server -- sudo tail -n 50 /var/log/cloud-init-output.log
```

These logs help identify errors during provisioning.

**2. Check application service logs**

```bash
multipass exec testpress-server -- sudo journalctl -u testpress-app.service -n 50 --no-pager
```

This displays the latest logs from the systemd service.

**3. Check Docker container logs**

```bash
multipass exec testpress-server -- sudo -u testpress docker logs testpress-app
```

This displays the application logs from the Docker container.

**4. Check Docker service status**

```bash
multipass exec testpress-server -- sudo systemctl status docker
```

This helps identify whether the Docker service is running correctly.


## Cloud-Init Execution Stages

Cloud-init runs in different stages during the VM boot process. Each stage performs a specific set of tasks.

### 1. Detect Stage

Cloud-init identifies the environment and determines which data source provides the configuration.

For this exercise, Multipass provides the cloud-init configuration when creating the VM.

### 2. Local Stage

This stage runs early during boot, before network configuration is fully available.

It identifies the data source and prepares the system for the remaining cloud-init stages.

### 3. Network Stage

This stage runs after the VM's network is configured.

It makes sure the VM can connect to the network and access external resources.

For example, cloud-init needs network access to download and install packages such as Docker, Git, and FFmpeg.

### 4. Config Stage

This stage processes most of the configuration provided in `cloud-init.yaml`.

For this exercise, it handles tasks such as:

- Creating users and groups.
- Writing application files.
- Updating the package index.
- Installing required packages.

### 5. Final Stage

This is the last stage of cloud-init execution.

For this exercise, it executes the commands defined under `runcmd`, including:

- Setting application directory ownership.
- Building the Docker image.
- Reloading systemd.
- Enabling and starting the application service.

It also displays the final provisioning message.

### Execution Frequency

Cloud-init stages are processed during each boot, but individual modules have different execution frequencies.

| Execution type | Behaviour |
|---|---|
| Once per instance | Runs during the first boot of a VM instance. |
| Per boot | Runs every time the VM boots. |
| Always | Runs every time the corresponding module is executed. |

Most provisioning tasks in this exercise are designed to run once per instance. Rebooting the VM does not normally repeat package installation, user creation, or the `runcmd` commands.

Cloud-init uses instance information and execution records stored under `/var/lib/cloud/` to determine which tasks have already been completed.


## Cloud-Init vs Dockerfile

Cloud-init and Dockerfiles serve different purposes.

**Cloud-init** is used to configure the virtual machine or server. It handles tasks such as installing packages, creating users, configuring services, and preparing the server environment.

**Dockerfile** is used to build a Docker image. It defines the application environment, required dependencies, files, and commands needed to run the application inside a container.

### Key Differences

| Cloud-init | Dockerfile |
|---|---|
| Configures the virtual machine or server. | Builds a Docker image. |
| Installs server-level packages and tools. | Defines the application environment and dependencies. |
| Creates users and groups on the server. | Can create users inside the container. |
| Configures systemd services. | Defines how the container starts. |
| Runs during VM provisioning. | Runs when building the Docker image. |

In simple terms, **cloud-init prepares the server, while the Dockerfile prepares the application environment inside the container.**