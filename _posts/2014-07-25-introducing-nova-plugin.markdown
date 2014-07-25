---
layout: blog_post
section: blog
---

The [imagefactory][] approach to image creation has always been to use the native installer for an OS to create an image that has _"Just Enough OS"_, customize that image by adding packages or running commands to create a base image, and then translating that base image into various formats compatible with a wide range of cloud/virtualization targets. These activities of creating the JEOS image, customizing, and converting are delegated out to plugins by imagefactory. While there have been a number of "cloud plugins" that allow imagefactory to work with numerous hypervisors and public clouds, there has only been one "OS plugin" and that is the TinMan plugin.

The name comes from the plugin's dependency on the [Oz][] project to create the JEOS image. Oz is a powerful tool that has been stable, fast, and offers support for a wide range of Linux distributions along with limited support for creating Microsoft Windows images. Oz has worked so well that TinMan has been the only OS plugin for imagefactory since the plugin API was added to imagefactory in early 2012.

While the Oz and TinMan have served us well, this combination has a couple of requirements the system running imagefactory must meet. Because Oz uses [libvirt][] and and TinMan uses [libguestfs][], imagefactory has been limited to running on systems that support these libraries. It also means that the system running imagefactory needs a fair amount of memory and storage to run installers in virtual machines and to store the images created. 

It's true that imagefactory has the ability to talk to a remote imagefactory and have the remote instance do the heavy lifting. But we live in exciting times when projects like [OpenStack][] have made it relatively easy to spin up our own clouds to pool together physical resources. With that available to us, why setup a system dedicated to building images? Using OpenStack directly allows us to break the tight dependency on libvirt and libguestfs and open up the possibility of running imagefactory on systems with fewer resources or systems that do not support those libraries. It also gives us a way to scale imagefactory for a higher number of concurrent builds by letting OpenStack manage the pool of physical resources.

####Introducing the Nova plugin for imagefactory:

The Nova plugin was written to allow imagefactory to benefit from access to an OpenStack infrastructure. Now, imagefactory can spawn an instance on OpenStack to run the OS installer. Once the JEOS installation is completed, the customization and creation of the base image is also done within OpenStack. The resulting base image can be used to create other OpenStack instances or it can be passed to one of the imagefactory Cloud plugins to be used on a different cloud or hypervisor.

The Nova plugin is a drop in replacement for the TinMan plugin. It handles the same OS, version, architecture combinations. In order to use Nova instead of TinMan, you'll want to remove the TinMan.info alias in /etc/imagefactory/plugins.d and make sure that Nova.info exists.

Once installed, using the Nova plugin is mostly the same as building images with the TinMan plugin with some additional arguments and setup related to connecting to OpenStack.

You'll want to make sure that you have the common OpenStack environment variables set where you are running imagefactory. These are `OS_AUTH_URL`, `OS_TENANT_NAME`, `OS_USERNAME`, and `OS_PASSWORD`.

If instances in the OpenStack you are connecting to require the instance to request a floating IP address instead of automatically assigning one, you'll want to pass the `request_floating_ip` option to imagefactory.

Similarly, if instances have a default non-root user, you need to tell imagefactory what that username is using the option `default_user` as well as how to run commands as a privileged user using the `command_prefix` option.

Finally, you need to choose what instance flavor to use for installation and customization steps. The flavor id can be passed to imagefactory with the `flavor` option.

####Example:

So, let's run through a quick example of building a base image using the Nova plugin. We'll build a Fedora 19 for x86_64 using the following kickstart and TDL files.

We'll use this kickstart file to setup the instance and specify the repository to use for installing over the network:

    url --url=http://download.fedoraproject.org/pub/fedora/linux/releases/19/Fedora/x86_64/os
    # Without the Everything repo, we cannot install cloud-init
    repo --name="fedora-everything" --baseurl=http://download.fedoraproject.org/pub/fedora/linux/releases/19/Everything/x86_64/os/
    install
    text
    keyboard us
    lang en_US.UTF-8
    skipx
    network --device eth0 --bootproto dhcp
    rootpw myrootpw
    firewall --disabled
    authconfig --enableshadow --enablemd5
    selinux --enforcing
    timezone --utc America/New_York
    bootloader --location=mbr
    zerombr
    clearpart --all --drives=vda

    part biosboot --fstype=biosboot --size=1 --ondisk=vda
    part /boot --fstype ext4 --size=200 --ondisk=vda
    part pv.2 --size=1 --grow --ondisk=vda
    volgroup VolGroup00 --pesize=32768 pv.2
    logvol swap --fstype swap --name=LogVol01 --vgname=VolGroup00 --size=768 --grow --maxsize=1536
    logvol / --fstype ext4 --name=LogVol00 --vgname=VolGroup00 --size=1024 --grow
    poweroff

    bootloader --location=mbr --timeout=5 --append="rhgb quiet"

    %packages
    @core
    cloud-init

    %end

This simple TDL file just specifies what OS and version we're building in order to select the correct OS plugin. 

    <template>
      <name>f19</name>
      <os>
        <name>Fedora</name>
        <version>19</version>
        <arch>x86_64</arch>
        <rootpw>test</rootpw>
        <install type='url'>
          <url>http://download.fedoraproject.org/pub/fedora/linux/releases/19/Fedora/x86_64/os</url>
        </install>
      </os>
      <disk>
        <size>20</size>
      </disk>
    </template>


We'll pass those two files to imagefactory using a command like this:

    imagefactory base_image \
     --file-parameter install_script f19.ks \
     --parameter request_floating_ip True \
     --parameter default_user ec2-user \
     --parameter command_prefix sudo \
     --parameter flavor 2 f19.tdl

A number of steps are completed as an image is built:

* First any images needed to boot into the installation are created or downloaded.

* Once a bare bones installation is completed, the instance is shutdown and a snapshot is taken before terminating the instance.

* A new instance is spawned from the snapshot and imagefactory installs any packages and/or runs any commands specified in the TDL by issuing commands via a series of ssh connections.

* Once customization is complete, an ICICLE document is created that lists all of the packages installed before shutting down this second instance.

* A snapshot is taken before terminating the instance.

Upon completion of these steps, imagefactory will report that the image has been created successfully and provide the UUID for reference. We can then pass this UUID when creating a target image for another cloud or hypervisor or we can use the final snapshot in OpenStack to spawn new instances.

Here is an example of the final output from imagefactory:

    ============ Final Image Details ============
    UUID: bdd32b97-0e94-473a-b376-74d2da289a2c
    Type: base_image
    Image filename: /var/lib/imagefactory/storage/bdd32b97-0e94-473a-b376-74d2da289a2c.body
    Image build completed SUCCESSFULLY!

####Getting the plugin:

The Nova plugin will be packaged and released with the 1.1.6 release of imagefactory. To start using this plugin now, pull the master branch from the imagefactory repository on GitHub.

To see a demo of the plugin being installed and used, have a look at this video:

[Introduction to the Nova plugin for Image Factory](http://youtu.be/UiJmF5iC8KI)

<div>
<iframe width="560" height="315" src="//www.youtube-nocookie.com/embed/UiJmF5iC8KI?rel=0" frameborder="0" allowfullscreen></iframe>
</div>


[imagefactory]: http://imgfac.org
[Oz]: https://github.com/clalancette/oz/wiki
[OpenStack]: http://www.openstack.org
[libvirt]: http://libvirt.org
[libguestfs]: http://libguestfs.org