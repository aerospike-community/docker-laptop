# Docker on Mac M1 with full x86_64 support

The below manual explains how to deploy docker with x86_64 support on Mac M1 processor. The system uses full CPU emulation.

On an Intel Mac, the system automatically uses Apple's Hypervisor instead, providing good virtualization speed.

## Version

This manual is for version 3. For instructions on v1, see the v1 branch.

## Version upgrade

This release (v3) must override v1 if previously used. Unfortunately, data between these 2 versions cannot be ported. Future releases will support updates without data/container loss.

## Note

These instructions and any files referenced within come with no warranty. Use at your own risk.

## Usage

### Delete previous installation, if it exists

```
cd ~
rm -rf .kubelab
```

### Install the KubeLab Virtual Machine

KubeLab is a pre-built Linux virtual machine with docker, and qemu wrapper scripts to allow for easy deployment.

```
cd ~
rm -f kubelab3.tgz
curl https://kubelab.s3.eu-west-1.amazonaws.com/kubelab3.tgz --output kubelab3.tgz
tar -zxvf kubelab3.tgz
cp -a ~/.kubelab/KubeLab-Launcher.app /Applications/
```

### Getting started

If you have `Automator` installed on your MacOS (this should be available by default), you can start KubeLab menu by running `KubeLab-Launcher` application as you normally would any other app.

Alternatively, you can start the menu by running the below script:

```
~/.kubelab/bin/main.sh
```

### Menu - Next steps

1. Create a data disk
2. Resize the data disk to your needs
3. Adjust CPU/RAM
4. Start KubeLab - note that after the first-time start, you may need to restart the Terminal app before docker becomes available (`Cmd+q`). This is because `DOCKER_HOST` variable gets added to your shell's rc file and needs to be loaded.
5. Stop KubeLab

### Advanced usage - Volumes and Snapshots

The VM uses two volumes:
 * root located under `~/.kubelab/images/root/`
 * data located under `~/.kubelab/images/data/`

The volume operations can be performed from the menu. See below for information about volume and snapshot use.

#### ROOT volume

The root volume has an underlying read-only image `ro.qcow2` and a writeable image `rw.qcow2`. The read-only image is used as a base, and any changes that happen to the root volume are written to `rw.qcow2`. As such, resetting the `rw` root volume from the menu will result in the OS of the VM to come back to it's initial state.

It is possible to save the new root volume (for example after running the `upgrade` option to upgrade the VM to the latest docker version). This merges the `rw.qcow2` image into `ro.qcow2`, essentially creating a new root base read-only volume. Once commited, you cannot revert back to the previous state. To do so, you would need to download the installation tgz again, and extract the original root `ro.qcow2` from it.

#### DATA volume

The data volume by default uses `rw.qcow2` read-write image. This image holds all the docker volumes and containers. It is possible to save the state of the volumes and containers to later use as a base data image. When commited, the `rw.qcow2` becomes a read-only base image called `ro.qcow2` and a new linked image `rw.qcow2` is created. It is then possible to revert back to the `ro.qcow2` image at any point. Kind of like creating a snapshot.

To illustrate this, see the following scenario:
1. a new data volume is created. This is `rw.qcow2`
2. kubelab is started, and you create a docker volume and container you intend to use every time for all tests
3. kubelab is stopped, and you commit the data volume to the `ro.qcow2` base image.
4. you start kubelab, perform testing, and stop kubelab
5. now you want to reset kubelab docker volumes and containers to the saved state. Just reset the rw data image, and this reverts to your `ro.qcow2` snapshot.

It is also possible to reset all data volumes, which will bring the system back to an empty docker.

## Running Aerospike

To easily deploy Aerospike on a lab environment, you can use [aerolab](https://github.com/aerospike/aerolab)

Alternatively, for a manual installation, follow [these instructions](https://docs.aerospike.com/server/operations/install/docker-desktop#start-a-container-and-the-database-by-using-this-image) to setup and deploy Aerospike.

## Manual non-menu usage instructions

It is possible to use the KubeLab setup without the menu script. For this, explore the scripts under `~/.kubelab/bin` and configuration parameters under `~/.kubelab/etc`

## Troubleshooting

To submit a bug report, please open a GitHub issue with a description of the problem, also providing `~/.kubelab/log/vm.log`
