# Yocto Environment for Tarragon and EVAcharge SE platforms

This is a wrapper repository that allows you to create a customized Linux root filesystem for EV charging infrastructure based on the open-source software stack **EVerest** https://github.com/EVerest/EVerest and chargebyte's hardware platform **Tarragon**.
For problems and inquiries: https://tickets.in-tech-smartcharging.com/servicedesk

## Table of Contents

1. [Introduction](#introduction)  
2. [Background](#background)  
2.1 [Layers](#layers)  
2.2 ["Wrapper" Repository](#wrapper)  
3. [Build with Yocto](#building)  
3.1 [System Requirements](#SystemRequirements)  
3.2 [Setting up the Yocto build environment](#Setting)  
3.3 [Adding or removing layers](#addorremove)  
3.4 [Building an Image](#build)  
3.5 [Building a firmware update image with rauc framework](#rauc-update)  
4. [Appendix](#appendix)  
A.1 [How to change kernel configurations](#kernel)  


## Introduction <a name="introduction"></a>

This document helps you to get started with creating a Linux root filesystem based on board support package (BSP) of Tarragon - the hardware platform offered by chargebyte GmbH for EV charging infrastructure and the open-source software stack EVerest. The document defines what the layers included in this Yocto Project are, and how you can use them to create a basic Linux distribution, which you can then extend by adding further packages specific to your application.

If you are new to Yocto, it is recommended to read the [Yocto Overview and Concepts Manual](https://docs.yoctoproject.org/overview-manual/index.html). To get a quick introduction to Yocto, this [Software Overview](https://www.yoctoproject.org/software-overview) might be helpful. For further documentation on the Yocto Project, including information about dealing with BSP layers and working with the Yocto Project's build system **BitBake**, check the [Yocto Project Documentation](https://docs.yoctoproject.org/).

## Background <a name="background"></a>

### Layers <a name="layers"></a>

As the Yocto Project is based on the concept of [layers](https://docs.yoctoproject.org/dev-manual/common-tasks.html#understanding-and-creating-layers), the following table lists all the layers used to create a basic distribution based on chargebyte’s charge control platforms Tarragon and EVAcharge SE.

| Layer | Description | Repository |
|--|--|--|
| meta-chargebyte | BSP layer for Tarragon & EVAcharge SE | https://github.com/chargebyte/meta-chargebyte |
| meta-chargebyte-distro | Distribution adaptations layer | https://github.com/chargebyte/meta-chargebyte-distro |
| meta-freescale | Layer containing NXP hardware support metadata | https://git.yoctoproject.org/cgit/cgit.cgi/meta-freescale |
| meta-openembedded | Collection of layers to supplement OE-Core with additional packages | https://github.com/openembedded/meta-openembedded |
| meta-rauc| Layer controlling and performing secure software updates for embedded Linux | https://github.com/rauc/meta-rauc |
| meta-everest | Layer containing EVerest charging stack | https://github.com/EVerest/meta-everest |
| meta-chargebyte-everest | Layer containing EVerest adjustments by chargebyte | https://github.com/chargebyte/meta-chargebyte-everest |
| poky | Build tool and metadata included in a reference distribution | https://git.yoctoproject.org/poky |

This layering approach increases flexibility to expand your project. You can add layers, which in turn would add packages essential for the distribution you want to build. Layers are usually available as repositories. Information on how to include or remove layers will be given in [Section 3.3](#addorremove). Note that you would still need to create a firmware bundle for the Linux distribution created by this setup, as the output is only a root filesystem in an `ext4`. By doing that, you would be able to easily update the firmware on the Tarragon board using e.g., RAUC. Instructions about how to use the resulting `ext4` to create a firmware bundle are included in our Charge Control C user guide. Contact us to get the latest version of it.

### "Wrapper" Repository <a name="wrapper"></a>

This "wrapper" repository has been created to facilitate downloading the above-mentioned layers/repositories on your local machine. It contains a manifest file called `default.xml` that holds information about the repositories representing the layers needed to build a distribution, and which branches to be checked out from these repositories. The process of extracting information from the manifest file and checking out the branches of the mentioned repositories is done by using `repo` commands. Installation and usage of the `repo` utility will be explained in [Section 3.2](#Setting). The manifest file for this repository looks like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <default sync-j="4" revision="kirkstone"/>

  <!-- remote repository definitions -->
  <remote fetch="https://git.yoctoproject.org/git" name="yocto"/>
  <remote fetch="https://github.com/openembedded" name="oe"/>
  <remote fetch="https://github.com/chargebyte" name="chargebyte"/>
  <remote fetch="https://github.com/rauc" name="rauc"/>
  <remote fetch="https://github.com/EVerest" name="everest"/>

  <!-- project definitions -->
  <project remote="yocto"        revision="kirkstone"                                name="poky"                    path="source"/>
  <project remote="yocto"        revision="kirkstone"                                name="meta-freescale"          path="source/meta-freescale"/>
  <project remote="oe"           revision="kirkstone"                                name="meta-openembedded"       path="source/meta-openembedded"/>
  <project remote="chargebyte"   revision="kirkstone"                                name="meta-chargebyte"         path="source/meta-chargebyte"/>
  <project remote="chargebyte"   revision="kirkstone"                                name="meta-chargebyte-distro"  path="source/meta-chargebyte-distro"/>
  <project remote="rauc"         revision="kirkstone"                                name="meta-rauc"               path="source/meta-rauc"/>
  <project remote="everest"      revision="kirkstone"                                name="meta-everest"            path="source/meta-everest"/>
  <project remote="chargebyte"   revision="kirkstone"                                name="meta-chargebyte-everest" path="source/meta-chargebyte-everest"/>
  <project remote="chargebyte"   revision="kirkstone-everest"                        name="chargebyte-bsp"          path="chargebyte-bsp">
    <linkfile dest="build/conf" src="conf"/>
  </project>

</manifest>
```

The attributes `revision` and `path` represent the source branch and the destination where the branch´s content will be placed on the local machine, respectively. You can set the `revision` to be either a branch name, by which you can continuously receive branch updates, or a particular commit in the repository. The tag `linkfile` links two folders together. It has two attributes; one is `src` which is a folder in the repository, and the other is `dest` representing the destination folder on the local machine, which will always have the content of the source folder i.e., mirror it.

Apart from the manifest file, the repository also has a configuration folder. This folder contains a `local.conf` file which contains variables and machine configurations you can alter to affect the resulting distribution and a `bblayers.conf` file which gives information about the layers included for building. Adding or removing layers can be done through this file, provided that the added layer, e.g., the cloned repository, is found on the local machine in the path described in this file.

## Build with Yocto <a name="building"></a>

### System Requirements <a name="SystemRequirements"></a>

Some packages are required by the build host to be able to cover all build scenarios using the Yocto Project. In this section of the [Yocto Reference Manual](https://docs.yoctoproject.org/ref-manual/system-requirements.html#required-packages-for-the-build-host) you can find some helpful instructions based on the Linux distribution you are using. If you are using a host other than Linux, this section of the [Yocto Project Development Tasks Manual](https://docs.yoctoproject.org/dev-manual/start.html#preparing-the-build-host) can help you setting up your host system for using Yocto. Some other prerequisites might be needed to build EVerest. These can be found in the [everest-core](https://github.com/EVerest/everest-core#readme) repository.

### Setting up the Yocto build environment <a name="Setting"></a>

To be able to build an image with Yocto, the following setup should be followed:

1. To make use of the manifest file, you have to install `repo` to get your Yocto environment ready. The `repo` utility was originally created to ease Android development. It makes it easy to reference several Git repositories within a top-level project, which you can then clone to your local machine all at once.

```bash
mkdir ~/bin
curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

You need to also make sure that `~/bin` is added to your `PATH` variable (Usually the directory is added automatically in Ubuntu).

```bash
echo 'export PATH="$PATH":~/bin' >> ~/.bashrc
```

2. Now you can use the `repo` tool to check out all the repositories listed in the manifest file.

```bash
mkdir yocto
cd yocto
repo init -u https://github.com/chargebyte/chargebyte-bsp -b kirkstone-everest
repo sync
```

After the command `repo sync` is executed, you should be able to find three folders in the created `yocto` directory:
1. `source`: Where all the repositories representing the layers are cloned.
2. `chargebyte-bsp`: A clone of the 'wrapper' repository containing the manifest file and configurations folder.
3. `build`: A link to the `conf` folder in `chargebyte-bsp`

### Adding or removing layers <a name="addorremove"></a>

Adding a layer can be done either:

**Manually**
1. Download the layer as a tarball or by cloning a repository.
2. Copy the folder where the layer is downloaded.
3. Paste it in the `yocto/source` folder.
4. Edit the `yocto/build/conf/bblayers.conf` file to include the new layer with the right path.

or **Automatically**
1. Edit the manifest file `chargebyte-bsp/default.xml`, so that it contains information about the source of the new layer
2. Issue the command `repo sync`. Note that all other layers will be synced as well, so any unsaved changes to your layers will be lost.
3. Edit the `yocto/build/conf/bblayers.conf` file to include the new layer with the right path.

You can then execute the command `bitbake-layers show-layers` to make sure that the new layer is now included.

To remove a layer, you can simply alter the `bblayers.conf` file by removing the layer path to make sure that the build system does not consider this layer while generating an image.

### Building an image <a name="build"></a>

To correctly set configurations related to the hardware platforms Tarragon and EVAcharge SE, the following table gives you insight about the images you can build:

| **`MACHINE`** | **`PROJECT`** | **`CUSTOMER`** | Resulting image |
|--|--|--|--|
| `tarragon` | `bsp` | `""` | Basic BSP image for Tarragon |
| `tarragon` | `bsp` | `developer`[^1] | BSP image for Tarragon with additional developer packages |
| `evachargese` | `bsp` | `""` | Basic BSP image for EVAcharge SE |
| `evachargese` | `bsp` | `developer` | BSP image for EVAcharge SE with additional developer packages |

For building an image, you would need to do the following:
1. Set the configurations for your build as mentioned in the table above. You can either:
  - Execute the following commands to e.g., set the machine to `tarragon` and project to `bsp`:
```bash
export MACHINE=tarragon
export PROJECT=bsp
export BB_ENV_PASSTHROUGH_ADDITIONS="PROJECT MACHINE"
```
  - Edit `yocto/build/conf/local.conf` directly. e.g., `MACHINE=...`.
2. Execute `source yocto/source/oe-init-build-env build` which initializes the build environment and changes the directory to `yocto/build`.
3. Execute `bitbake core-image-minimal` to build the image.

The resulting image will be found in `yocto/build/tmp/deploy/image/<machine>`.

### Building a firmware update image with rauc framework  <a name="rauc-update"></a>

The chargebyte's meta-chargebyte-everest layer is prepared for building a firmware update image using the rauc framework.

If you don't want to fine-tune the update image further, the only remaining steps are:
1. Create a firmware signing key if not already done. For this, we kindly refer to the good [rauc manual](https://rauc.readthedocs.io/).
2. Provide the key and certificate location to the Yocto build environment. This can be done in different ways, e.g.,  
  - extend your `yocto/build/conf/local.conf`, or
  - use a `core-bundle.bbappend` file to extend the existing Bitbake recipe.  
  In either variant, the file content must look like:
```
RAUC_KEY_FILE  = "/path/to/your/signing.key"
RAUC_CERT_FILE = "/path/to/your/signing.crt"
```
3. Finally, execute `bitbake core-bundle` to build the firmware update image. The resulting image can be found in
   `yocto/build/tmp/deploy/image/<machine>` and is named by default `EVerest-Firmware...image`.

You can transfer this file to the target device and install it e.g. using `rauc install EVerest-Firmware...image`.

## Appendix <a name="appendix"></a>

### How to change kernel configurations <a name="kernel"></a>

It may occur that you want to adapt the system's kernel configuration to be suitable for your special requirements, e.g., adding drivers or packages like Docker. Here are the steps you need to follow to generate a Linux image based on your desired kernel configuration:

1. Execute the command `bitbake linux-imx -c menuconfig` to open a tool that permits you to change the kernel configuration.
2. If you know the configuration name by the convention `CONFIG_*` only, you can search for it by typing "/" and putting its name after. You will be told which other configurations you need to set in order to be able to view this configuration and set it, as sometimes they may be hidden. Type "z" to find the hidden configurations. Press exit and choose "yes" to save the configurations temporarily.
3. You will need to perform the following commands in your build directory to save the configuration permanently and move them to your recipe folder.
```bash
bitbake -c savedefconfig linux-imx

cp tmp/work/tarragon-poky-linux-gnueabi/linux-imx/4.9.123-r0/build/defconfig  ../source/meta-chargebyte/recipes-kernel/linux/linux-imx/imx/
```
4. In the bbappend file found in meta-chargebyte/recipes-kernel/linux/linux-imx_%.bbappend, change the SRC_URI to be the following:
```bash
SRC_URI = "\
            git://github.com/chargebyte/linux.git;protocol=https;branch=${SRCBRANCH} \
            file://defconfig \
          "
```
5. Perform the following commands to remove the output files and shared state cache for the linux kernel, and then build a new image with the new kernel configuration.
```bash
bitbake -c cleansstate linux-imx

bitbake core-image-minimal
```
6. You may check the .config file located in tmp/work/tarragon-poky-linux-gnueabi/linux-imx/4.9.123-r0/build/ to make sure that your changes were added to the kernel configuration.

[^1]: to be able to build a developer image, check the respective documentation in `local.conf`
