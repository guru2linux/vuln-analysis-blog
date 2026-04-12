---
title: "Setting Up Wazuh on a Proxmox Server"
date: 2026-04-11
layout: default
---

# Setting Up Wazuh on a Proxmox Server

Wazuh is a powerful open-source security platform that provides unified security monitoring and management. In this guide, we will walk through the steps to set up Wazuh on a Proxmox server.

## Prerequisites

Before starting, ensure you have the following:

- A Proxmox server up and running.
- Sufficient resources (CPU, RAM, and storage) for the Wazuh server.
- Access to the Proxmox web interface.
- Basic knowledge of Linux commands.

## Step 1: Create a Virtual Machine for Wazuh

1. Log in to the Proxmox web interface.
2. Navigate to **Datacenter > Node > Create VM**.
3. Fill in the following details:
   - **VM ID**: Choose a unique ID.
   - **Name**: Enter a name for the VM (e.g., `Wazuh-Server`).
4. Configure the hardware:
   - **CPU**: Assign at least 2 cores.
   - **RAM**: Allocate at least 4GB of memory.
   - **Disk**: Assign at least 20GB of storage.
5. Select the installation ISO for your preferred Linux distribution (e.g., Ubuntu Server).
6. Complete the VM creation process.

## Step 2: Install the Operating System

1. Start the newly created VM.
2. Open the console and follow the installation steps for your chosen Linux distribution.
3. Update the system packages after installation:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

## Step 3: Install Docker (Optional)

Wazuh can be installed using Docker for easier management. To install Docker:

1. Install required packages:
   ```bash
   sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
   ```
2. Add Docker’s official GPG key:
   ```bash
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   ```
3. Add the Docker repository:
   ```bash
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```
4. Install Docker:
   ```bash
   sudo apt update && sudo apt install docker-ce docker-ce-cli containerd.io -y
   ```
5. Verify Docker installation:
   ```bash
   docker --version
   ```

## Step 4: Install Wazuh

1. Follow the official Wazuh installation guide for your setup:
   - [Wazuh All-in-One Installation](https://documentation.wazuh.com/current/deployment-options/all-in-one-deployment.html)
   - [Wazuh Cluster Installation](https://documentation.wazuh.com/current/deployment-options/cluster-deployment.html)
2. If using Docker, pull the Wazuh Docker image:
   ```bash
   docker pull wazuh/wazuh
   ```
3. Run the Wazuh container:
   ```bash
   docker run -d --name wazuh -p 1514:1514/udp -p 55000:55000 wazuh/wazuh
   ```

## Step 5: Access the Wazuh Dashboard

1. Open a web browser and navigate to the Wazuh dashboard URL.
2. Log in using the default credentials provided during installation.
3. Configure your agents and start monitoring.

## Conclusion

You have successfully set up Wazuh on a Proxmox server. Wazuh provides a comprehensive security monitoring solution, and with this setup, you can start securing your infrastructure effectively.