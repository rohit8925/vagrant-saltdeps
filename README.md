# vagrant-saltdeps
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/joshughes/vagrant-saltdeps?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

When testing salt formula with vagrant you may have one or many dependent formulas that are required to fully test your new formula. Vagrantsaltdeps is meant to help you manage these dependencies by checking them out from git and making them available to your vagrant machine.

**NOTE:** This plugin requires Vagrant 1.2+,

## Features

* Check out dependent salt formula from git
* Branch support for dependent formula
* Grains and Pillar merging from dependent formula


## Usage

Install using standard Vagrant 1.1+ plugin installation methods.

```
$ vagrant plugin install vagrant-saltdeps
...
```


## Quick Start

After installing the plugin (instructions above), you must create a saltdeps.yml file somewhere in your repo. This file along with your grains and pillar files must live in the same place across your repos. Here is an example.

```
name: test
deps:
  apache:
    git: git@github.com:saltstack-formulas/apache-formula.git
    branch: develop
```
* **name** - The name of your current formula, and the folder in the same directory as the Vagrantfile where your salt state data exists. This also determines the name of the folder created in `/srv/salt`
* **deps** - A list of dependencies for your formula
  * **apache** - In this example we are telling saltdeps that we expect a folder in this repo with our salt states to be called apache. This also determins the folder names that saltdeps creates
    * **git** - The git repo to checkout from. Currently only git ssh protocol is supported.
    * **branch** - The branch of the dependent fromula you would like to check out

After defining a saltdeps.yml file you must configure your Vagrantfile. You should put the vagrant-saltdeps provisioner before your salt provisioner since it touches the salt provisioner configuration.

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "ubuntu/trusty64"
  config.vm.synced_folder "./.vagrant-salt", "/srv/salt", id: "vagrant-root"

  config.ssh.forward_agent = true

  config.vm.provider :virtualbox do |vb|
    # Use VBoxManage to customize the VM. For example to change memory:
    vb.customize ["modifyvm", :id, "--memory", "1024"]
  end

  if Vagrant.has_plugin?("vagrant-saltdeps")
    config.vm.provision :saltdeps do |deps|
      deps.base_vagrantfile = "ssh://git@github.com/jhughes/vagrant-base.git"
      deps.checkout_path =  "./.vagrant-salt/deps"
      deps.deps_path     =  "./.vagrant-salt/saltdeps.yml"
    end
  end

  # Provision VM with current formula, in masterless mode
  config.vm.provision :salt do |salt|
    salt.minion_config = "./.vagrant-salt/minion"
    salt.run_highstate = true
    salt.install_type = 'stable'
    salt.colorize = true
    salt.verbose = true
  end

  # Run serverspec tests
  if Vagrant.has_plugin?("vagrant-serverspec")
    config.vm.provision :serverspec do |spec|
      spec.pattern = './spec/*_spec.rb'
    end
  end
end

```

This will get you started using the plugin. For further plugin configuration options please read below.

With the example above a new folder will be created in `.vagrant-salt/deps` called `apache-formula`. In this folder saltdeps will checkout the development branch from git and search to see if any grains or pillar files from that formula need to be merged into this formula's current grains and pillars. It will then configure the salt provisioner so it has access to the grain and pillar data and sync the apache formula to the vagrant vm.

## Configuration

This project exposes a few configuration options so you can decide the best way to lay out your salt-formula and still be able to test it in vagrant.

* **checkout_path** - Where you want saltdeps to checkout your dependent formulas. Defaults to `.saltdeps`
* **deps_path** - The path to your saltdeps.yml file. Defaults to `.vagrant-salt/saltdeps.yml'
* **grains_path** - The path in **all** your repos where the grains file can be found. Defaults to `.vagrant-salt/grains`
* **pillars_path** - The path in **all** your repos where the pillars file can be found. Defaults to `.vagrant-salt/pillars`
* **merge_pillars** - Tells saltdeps if it should merge your dependent pillars into one file. Defaults to `true`.
* **merge_grains** - Tells saltdeps if it should merge your dependent grains into one file. Defaults to `true`.
* **merge_path** - Path where saltdeps should put the results of the merged pillars and grains files. Defaults to `.vagrant-salt/compiled_grains` and `.vagrant-salt/compiled_pillars`.
* **base_vagrantfile** - This defines a git repo that has a single `Vagrantfile` in the root of the repository. This Vagrantfile is loaded before the Vagrantfile in the root of the project. The intention here is to allow for common configuration to be shared between salt formula. Any settings in the base Vagrantfile can be overridden with the Vagrantfile in your salt formula repository. The goal is to reduce the number of edits you have to do across the Vagrantfiles in your different formulas. 


## Development

To work on the `vagrant-saltdeps` plugin, fork this repository out, and use
[Bundler](http://gembundler.com) to get the dependencies: **Note:** vagrant requires `bundler < 1.8`. So you will need to install `1.7.13` and use the syntax below to make sure you are running with that version.

```
$  bundle _1.7.13_
```

Currently there are not unit tests, but to test you can go to a directory with a Vagrantfile and run the following.

```
$  bundle _1.7.13_ exec vagrant provision
```
