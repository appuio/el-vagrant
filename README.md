# Base Vagrantfile for Enterprise Linux

This repository contains a Vagrantfile for Red Hat Enterprise Linux (RHEL) and Community Enterprise Operating System (CentOS).
The Vagrantfile is designed to be included by other Vagrantfiles and takes care of the
following tasks:

* Configures machine time zone based on host time zone
* Configures `/etc/hosts` on machines so that they can look up each other
* Creates and deploys an SSH key so that machines can log in to each other (useful for Ansible)
* Registers and unregisters machines at Red Hat (RHEL only)

## Requirements

This Vagrantfile has been tested with Vagrant 2.0.0. It requires the following
plugins which it installs automatically as needed:

* [vagrant-timezone](https://github.com/tmatilai/vagrant-timezone)
* [vagrant-hostmanager](https://github.com/devopsgroup-io/vagrant-hostmanager)
* [vagrant-triggers](https://github.com/emyl/vagrant-triggers)

## Usage

This Vagrantfile is designed to be included by other Vagrantfiles. Assuming
your project is located in the same directory where you checked out el-vagrant.

    load '../el-vagrant/Vagrantfile'

Change the path accordingly if your project resides in a different location.

### Subscription Manager Configuration (RHEL only)

RHEL machines need a valid subscription for installing additional software packages. El-vagrant
automatically registers machines at Red Hat if the Suscription Manager is installed and
the following variables are present in the project Vagrantfile or user Vagrantfile (usually in ~/.vagrant.d/):

* `$rhsm_username` (required): Username used for registering machines with Red Hat Subscription Management
* `$rhsm_password` (required): Password used for registering machines with Red Hat Subscription Management
* `$rhsm_pool`: ID of a subscription pool attach to the machines. Auto-attachment is used
if this variable is not set.

Example:

    $rhsm_username = 'me@example.org'
    $rhsm_password = 'mypassword'
    $rhsm_pool = '2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a2a'

El-vagrant registers machines with system names of the form `vagrant-<hostname>-<machinename>` so that they are easily recognizable in Red Hat Subscription Management. Where `<hostname>` is the name of the host running Vagrant and `<machinename>` is the name of the Vagrant machine (guest).

## RHEL Box

Since RHEL boxes can't be freely distributed you have to build one yourself.
The [rhel-vagrant-box](https://github.com/appuio/rhel-vagrant-box) repository provides a script for creating RHEL
boxes for the [Libvirt provider](https://github.com/vagrant-libvirt/vagrant-libvirt).

## Features

### Hostname Resolution

El-vagrant configures hostname resolution with the help of the
[vagrant-hostmanager](https://github.com/devopsgroup-io/vagrant-hostmanager) plugin.
Vagrant-hostmanager manages entries in the `/etc/hosts` files for all running machines so
that they can look up each other.  
If you need hostname resolution via DNS you can install [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) in your Vagrant machines.
It reads `/etc/hosts` by default and makes its entries available through DNS.

### SSH Key Distribution

El-vagrant creates an SSH key in the directory in which your `Vagrantfile` resides and deploys this key so that
the `root` and `vagrant` users can log into other machines of the same project as `root` and `vagrant`. Among
other things this setup is useful for running Ansible on your Vagrant machines with or without privilege escalation (sudo).
