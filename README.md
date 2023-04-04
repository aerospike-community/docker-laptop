# docker-laptop

## Installation

```
cd ~
curl https://docker-laptop.s3.eu-west-1.amazonaws.com/docker-laptop.tgz --output docker-laptop.tgz
tar -zxvf docker-laptop.tgz
sudo cp -a ~/.docker-laptop/Docker\ Laptop.app /Applications/
rm -f docker-laptop.tgz
```

## Usage

If you have `Automator` installed on your MacOS (this should be available by default), you can start KubeLab menu by running `Docker Laptop` application as you normally would any other app.

Alternatively, you can start the menu by running the below script:

```
~/.docker-laptop/bin/menu.sh
```

## Running Aerospike

To easily deploy Aerospike on a lab environment, you can use [aerolab](https://github.com/aerospike/aerolab)

Alternatively, for a manual installation, follow [these instructions](https://docs.aerospike.com/server/operations/install/docker-desktop#start-a-container-and-the-database-by-using-this-image) to setup and deploy Aerospike.

## Manual non-menu usage instructions

It is possible to use the KubeLab setup without the menu script. For this, explore the scripts under `~/.docker-laptop/bin` and configuration parameters under `~/.docker-laptop/etc`

## Troubleshooting

To submit a bug report, please open a GitHub issue with a description of the problem, also providing `~/.docker-laptop/log/*`
