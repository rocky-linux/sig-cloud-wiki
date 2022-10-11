# Rocky Linux SIG/Cloud Wiki

The Rocky Linux Cloud Special Interest  Group is dedicated to the enablement of  of Rocky Linux on Cloud Services Providers in private and public clouds.

Anyone is welcome to join and participate in the activities and responsibilities of the Cloud SIG.

This documentation is a  work in progress and we welcome any and all contributions, which can be made on [git.resf.org/sig_core/wiki](https://git.resf.org/sig_core/wiki)

## Links

- [Peridot - Rocky Linux 9 - SIG Cloud](https://peridot.build.resf.org/15016370-1410-4459-a1a2-a1576041fd19)
- [Peridot - Rocky Linux 8 - SIG Cloud](https://peridot.build.resf.org/f91da90d-5bdb-4cf2-80ea-e07f8dae5a5c)
- [RPMS](https://git.rockylinux.org/sig/cloud/rpms)
- [Patches](https://git.rockylinux.org/sig/cloud/patch)
- [peridot-config](https://git.rockylinux.org/sig/cloud/peridot-config)

## Responsibilities

The Cloud SIG is mainly responsible for two items: Cloud Images and Optimized Kernels for clouds.

### Images

A multitude of images are created as part of the Rocky Linux release process, using [kickstarts](https://git.resf.org/sig_core/kickstarts) and Rocky Release Engineering's [toolkit](https://git.resf.org/sig_core/toolkit) "Empanadas" to compose the images.

Currently, the following images are produced by the SIG:

* Container rootfs (Base and Minimal variants)
* GenericCloud
* EC2 (Amazon AWS)
* Oracle Cloud Platform
* Google Cloud Platform (Images are generated [by GCP](https://github.com/GoogleCloudPlatform/compute-image-tools/))
* Microsoft Azure
* Vagrant-compatibile images (Libvirt, Virtualbox, VMWare)
    * NOTE(nhanlon) 2022-10-11: These images are currently a best-effort production until the workflows are automated to create these images.


### Optimized Kernels

The Cloud SIG also produces a custom kernel artifact for use on various cloud providers. Currently, the major changes in this kernel are the backporting of several changes from stable/mainline kernel branches relating to GVNic drivers for improved performance on Google Cloud Platform. Note that the Kernel is the same version as in any given Rocky Release, but has patches from kernel.org branches backported and applied on top.

A full description of the changes the this Kernel has can be found [here](packages/kernel.md).

Any and all changes which may benefit ease or performance of usage of Rocky on clouds are welcome to be contributed. In order to maintain patches, a tool called [Sideline](https://github.com/rocky-linux/sideline) was developed to aid in grabbing and applying patches to the Rocky Linux standard Kernel.

## Members

* Neil Hanlon
* Brian Clemens
* Mustafa Gezen
* Zach Marano
* Gregory Kurtzer
* Dave Linksey

## Project layout

    mkdocs.yml    # The configuration file.
    docs/
        index.md  # The documentation homepage.
        ...       # Other markdown pages, images and other files.
