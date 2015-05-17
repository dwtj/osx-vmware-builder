# Packer Template for Building VMware OS X Boxes

This is a Packer template and a set of support scripts that will prepare an OS X installer media that performs an unattended install for use with [Packer](http://packer.io). This is a customized fork of Timothy Sutton's excellent work on `timsutton/osx-vm-templates`. That project has support for `VeeWee` and two additional providers.

The machine defaults to being configured for use with [Vagrant](http://www.vagrantup.com), and supports one provider: the [Hashicorp VMware Fusion provider](http://www.vagrantup.com/vmware)

It's possible to build a machine with different admin account settings, and without the vagrant ssh keys, for use with other systems such as CI.

Use with the Fusion provider requires Vagrant 1.3.0, and use with the VirtualBox provider Vagrant 1.6.3 if using the Rsync file sync mechanism.

Provisioning steps that are defined in the template via items in the [scripts](https://github.com/dwtj/osx-vmware-builder/tree/master/scripts) directory:
- [Vagrant-specific configuration](http://docs.vagrantup.com/v2/boxes/base.html)
- VM guest tools installation if on VMware
- Xcode CLI tools installation


## Preparing the ISO

OS X's installer cannot be bootstrapped as easily as can Linux or Windows, and so exists the [prepare_iso.sh](https://github.com/timsutton/osx-vm-templates/blob/master/prepare_iso/prepare_iso.sh) script to perform modifications to it that will allow for an automated install and ultimately allow Packer and later, Vagrant, to have SSH access.

Run the `prepare_iso.sh` script with two arguments: the path to an `Install OS X.app` or the `InstallESD.dmg` contained within, and an output directory. Root privileges are required in order to write a new DMG with the correct file ownerships. For example, with a 10.8.4 Mountain Lion installer:

`sudo prepare_iso/prepare_iso.sh "/Applications/Install OS X Mountain Lion.app" out`

...should output progress information ending in something this:

```
-- MD5: dc93ded64396574897a5f41d6dd7066c
-- Done. Built image is located at out/OSX_InstallESD_10.8.4_12E55.dmg. Add this iso and its checksum to your template.
```

`prepare_iso.sh` also accepts three command line switches to modify the details of the admin user installed by the script.

* `-u` modifies the name of the admin account, defaulting to `vagrant`
* `-p` modifies the password of the same account, defaulting to `vagrant`
* `-i` sets the path of the account's avatar image, defaulting to `prepare_iso/support/vagrant.jpg`

For example:

`sudo prepare_iso/prepare_iso.sh -u admin -p password -i /path/to/image.jpg "/Applications/Install OS X Mountain Lion.app" out`

#### Clone this repository

The `prepare_iso.sh` script needs the `support` directory and its content. In other words, the easiest way to run the script is after cloning this repository.

#### Snow Leopard

The `prepare_iso.sh` script depends on `pkgbuild` utility. As `pkgbuild` is not installed on Snow Leopard (contrary to the later OS X), you need to install XCode 3.2.6 which includes it.

## Use with Packer

The path and checksum can now be added to your Packer template or provided as [user variables](http://www.packer.io/docs/templates/user-variables.html). The `packer` directory contains a template that can be used with the `vmware-iso` builder.

The Packer template adds some additional VM options required for OS X guests. Note that the paths given in the Packer template's `iso_url` builder key accepts file paths, both absolute and relative (to the current working directory).

Given the above output, we could run then run packer:

```sh
cd packer
packer build \
  -var iso_checksum=dc93ded64396574897a5f41d6dd7066c \
  -var iso_url=../out/OSX_InstallESD_10.8.4_12E55.dmg \
  template.json
```

You might also use the `-only` option to restrict to either the `vmware-iso` or `virtualbox-iso` builders.

If you modified the name or password of the admin account in the `prepare_iso` stage, you'll need to pass in the modified details as packer variables. You can also prevent the vagrant SSH keys from being installed for that user.

For example:

```
packer build \
  -var iso_checksum=dc93ded64396574897a5f41d6dd7066c \
  -var iso_url=../out/OSX_InstallESD_10.8.4_12E55.dmg \
  -var username=youruser \
  -var password=yourpassword \
  -var install_vagrant_keys=false \
  template.json
```


## Automated installs on OS X

OS X's installer supports a kind of bootstrap install functionality similar to Linux and Windows, however it must be invoked using pre-existing files placed on the booted installation media. This approach is roughly equivalent to that used by Apple's System Image Utility for deploying automated OS X installations and image restoration.

The `prepare_iso.sh` script in this repo takes care of mounting and modifying a vanilla OS X installer downloaded from the Mac App Store. The resulting .dmg file and checksum can then be added to the Packer template. Because the preparation is done up front, no boot command sequences, attached devices or web server access is required.

More details as to the modifications to the installer media are provided in the comments of the script.


## Supported guest OS versions

Currently this prepare script and template supports OS X Lion (10.7) through Yosemite (10.10).


## Automated GUI logins

For some kinds of automated tasks, it may be necessary to have an active GUI login session (for example, test suites requiring a GUI, or Jenkins SSH slaves requiring a window server for their tasks). The Packer templates support enabling this automatically by using the `autologin` user variable, which can be set to `1` or `true`, for example:

`packer build -var autologin=true template.json`

This was easily made possible thanks to Per Olofsson's [CreateUserPkg](http://magervalp.github.com/CreateUserPkg) utility, which was used to help create the box's vagrant user in the `prepare_iso` script, and which also supports generating the magic kcpassword file with a particular hash format to set up the auto-login.

## System updates

Packer will instruct the system to download and install all available OS X updates, if you want to disable this default behaviour, use `update_system` variable:

```
packer build -var update_system=0 template.json
```


## Box sizes

It might be advisable to remove (with care) some unwanted applications in an additional postinstall script. It should also be possible to modify the OS X installer package to install fewer components, but this is non-trivial. One can also supply a custom "choice changes XML" file to modify the installer choices in a supported way, but from my testing, this only allows removing several auxiliary packages that make up no more than 6-8% of the installed footprint (for example, multilingual voices and dictionary files).


## Alternate approaches to VM provisioning

Mads Fog Albrechtslund documents an [interesting method](http://hazenet.dk/2013/07/17/creating-a-never-booted-os-x-template-in-vsphere-5-1) for converting unbooted .dmg images into VMDK files for use with ESXi.
