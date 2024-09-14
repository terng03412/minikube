# MiniKF: A Fast and Easy Way to Deploy Kubeflow on Your Laptop

MiniKF offers a production-ready, full-fledged, local Kubeflow deployment that installs in minutes. It's designed for quick experimentation and running complete Kubeflow Pipelines. When you're ready to scale up, you can move to a Kubeflow cloud deployment with one click, without having to rewrite anything.

For more information, please see the official announcement and the rationale behind MiniKF.

## Community Support

Join the discussion on the #minikf Slack channel to ask questions, request features, and get support for MiniKF. To join the Kubeflow Slack workspace, please request an invite.

## System Requirements

For a smooth experience, we recommend that your system meets the following requirements:
* 12GB RAM
* 2 CPUs
* 50GB disk space

## Supported Operating Systems

MiniKF runs on all major operating systems:
* Linux
* macOS
* Windows

## Prerequisites

Before installing MiniKF, you need to have the following software installed on your laptop:
* Vagrant
  - Install from: https://developer.hashicorp.com/vagrant/install
* VirtualBox
  - **Important**: MiniKF currently requires VirtualBox 7.0, as the latest version of MiniKube does not support VirtualBox 7.1.
  - Install from: https://www.virtualbox.org/wiki/Download_Old_Builds_7_0

## MiniKF Installation

1. Open a terminal on your laptop
2. Create a new directory and switch into it
3. Run the following commands to install MiniKF:

   ```
   vagrant init arrikto/minikf
   vagrant up
   ```

4. MiniKF will take a few minutes to boot. 
5. When the boot process is complete, navigate to http://10.10.10.10 in your web browser
6. Follow the on-screen instructions to start Kubeflow and Rok

## MiniKF Upgrade

If you're upgrading from a previous version, follow these step-by-step instructions:

1. Upgrade the MiniKF box to the latest version:
   ```
   vagrant box update
   ```

2. Ensure you have updated to the latest version:
   ```
   vagrant box list
   ```

3. Upgrade the `vagrant-persistent-storage` plugin to v0.0.47 or later:
   ```
   vagrant plugin update vagrant-persistent-storage
   ```

4. Destroy the VM:
   ```
   vagrant destroy
   ```

5. Remove all local state. This will remove all of your customization in MiniKF (notebooks, pipelines, Rok snapshots):
   * Windows: `del minikf-user-data.vdi`
   * Linux/macOS: `rm minikf-user-data.vdi`

6. Re-create your VM:
   ```
   vagrant up
   ```

Remember to back up any important data before performing an upgrade, as this process will remove all local customizations.
