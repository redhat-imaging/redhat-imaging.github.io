---
layout: blog_post
section: blog
---

Have you found yourself wishing that Image Factory supported your favorite tool for building system images or some new hypervisor / cloud? Maybe you're developing a new cloud offering or you have a favorite OS that you would like to see Image Factory build images for.

Would you be surprised if I told you that writing a plugin for Image Factory is relatively easy? Actually, let's back up a second... are you surprised to learn that Image Factory uses plugins? 

If the answer to any of these questions was 'Yes', then hopefully this post will introduce you to an exciting world of image building possibilities.

Since the Nova plugin was just released, I'll tear that open to use as a real world example. The source for this plugin can be found in the imagefactory git [repository][].

####What's in a plugin?

First, what IS an imagefactory plugin? If you look at the plugins already installed on your system, you'll see a number of directories that look like Python packages. You will find these in the `imagefactory_plugins` directory in the `SITE_PACKAGES` for the Python environment under which you run imagefactory.  

According to the imagefactory plugins [documentation][], there are only two things you need to add to a Python package in order to make it a plugin that imagefactory will recognize and load.

These tasks are:

1. Assign the delegate class.  
2. Define the plugin metadata.

Let's take a look at the second item first and create a .info document that describes our plugin. This file is named after the plugin it describes. In our case, this would be the `Nova.info` file that looks like this:

    {
        "type": "os",
        "targets": [ ["Fedora", null, null], ["RHEL-6", null, null], ["RHEL-5", null, null],
                     ["RHEL-7", null, null], ["Ubuntu", null, null], ["CentOS-6", null, null],
                     ["CentOS-5", null, null], ["Windows", null, null] ],
        "description": "Plugin to support building OS base images using OpenStack Nova.",
        "maintainer": {
            "name": "Red Hat, Inc.",
            "email": "imagefactory-devel@lists.fedorahosted.org",
            "url": "http://imgfac.org"
        },
        "version": "1.0",
        "license": "Copyright 2012 Red Hat, Inc. - http://www.apache.org/licenses/LICENSE-2.0"
    }

The .info file is just a JSON document that describes the plugin with a set of key/value pairs.See all of that stuff there? It is REALLY important that *everything* is defined before a plugin is released. Okay, now that I've said that, I'll point out that the only two keys that need values in order for the plugin to be loaded by imagefactory are the first two, `type` and `targets`.

The "type" of the plugin is either "os" or "cloud". If your plugin will be installing an operating system into an image in some way, then it is an "os" plugin. If your plugin will be transforming a base image that already has an operating system installed into something that your target hypervisor or cloud can use, then the plugin is a "cloud" plugin. This will come into play once we start writing code to implement our plugin, but we'll get to that soon enough.

The "targets" value is a list of terms matched when building a base or target image. In the case of an OS plugin, each value in that list is itself a list of the OS name, the OS version, and the OS architecture. In a TDL document, this matches the `name`, `version`, `arch` child elements under the `os` element. In our example the above, `null` is used as a wildcard meaning that each entry matches for any version or architecture of the OS listed by name.

For a Cloud plugin, each of the values in the targets list is just a string that is the name of the cloud or hypervisor with which the plugin is compatible. This cloud name is what will be passed to imagefactory when using the `target_image` or `provider_image` command. Here is the target entry in the .info file for the Docker plugin as an example:

        "targets": [ [ "docker"] ],

Now that we have described the plugin, we need to implement it. Here is where we need to satisfy the first requirement of assigning the delegate class. The delegate class is the interface imagefactory uses to interact with the plugin and define the methods your plugin can implement. This class will implement either the [OSDelegate][] or [CloudDelegate][] interface. The Nova plugin, the example for this article, is an OS plugin, so the delegate class implements the OSDelegate interface in the `Nova.py` module:

    import zope
    import logging

    class Nova(object):
        zope.interface.implements(OSDelegate)
        def __init__(self):
            super(Nova, self).__init__()
            self.log = logging.getLogger('%s.%s' % (__name__, self.__class__.__name__))

        def create_base_image(self, builder, template, parameters):
            self.log.info('create_base_image() called for Nova plugin - creating a BaseImage')

        def create_target_image(self, builder, target, base_image, parameters):
            self.log.info('create_target_image() called for Nova plugin - creating a TargetImage')

In order for imagefactory to find the class that implements the delegate interface, in this case `Nova`, we need to assign that class. We do this in the `__init__.py` of our plugin package with a line like this:

    from Nova import Nova as delegate_class

Now imagefactory knows how to call into our plugin when creating a base image for the operating systems our plugin supports.

Our example code listing above doesn't do anything but log that each method was called. There is, of course, more to connecting to OpenStack Nova to spawn an instance, install an OS, and customize it for use by future instances. Luckily, there is a project we can use in this plugin to handle these tasks. Once we have installed [Nova Image Builder][] we can import what we need into our delegate class and call out to this other project to do the heavy lifting for us:

    import zope
    import logging
    from novaimagebuilder.Builder import Builder

    class Nova(object):
        zope.interface.implements(OSDelegate)
        def __init__(self):
            super(Nova, self).__init__()
            self.log = logging.getLogger('%s.%s' % (__name__, self.__class__.__name__))
            self.nib = None

        def create_base_image(self, builder, template, parameters):
            self.log.info('create_base_image() called for Nova plugin - creating a BaseImage')
            if template.os_version:
                if template.os_name[-1].isdigit():
                    install_os = '%s.%s' % (template.os_name, template.os_version)
                else:
                    install_os = '%s%s' % (template.os_name, template.os_version)
            else:
                install_os = template.os_name

            install_os = install_os.lower()

            install_location = template.install_location
            # TDL uses 'url' but Nova Image Builder uses 'tree'
            install_type = 'tree' if template.install_type == 'url' else template.install_type
            install_script = parameters.get('install_script')
            install_config = {'admin_password': parameters.get('admin_password'),
                              'license_key': parameters.get('license_key'),
                              'arch': template.os_arch,
                              'disk_size': parameters.get('disk_size'),
                              'flavor': parameters.get('flavor'),
                              'storage': parameters.get('storage'),
                              'name': template.name,
                              'direct_boot': parameters.get('direct_boot', False),
                              'timeout': parameters.get('timeout', 1800),
                              'public': parameters.get('public', False),
                              'floating_ip': parameters.get('request_floating_ip', False)}

            builder.base_image.update(10, 'BUILDING', 'Created Nova Image Builder instance...')
            self.nib = Builder(install_os, install_location, install_type, install_script, install_config)
            self.nib.run()
            jeos_image_id = self.nib.wait_for_completion(180)
            builder.base_image.properties['x-image-properties-glance_id'] = jeos_image_id

In the code above, we are gathering the information Nova Image Builder will need from the TDL that was passed in along with any external build parameters. This information is then passed to a new instance of the Builder class in Nova Image Builder, which is then told to run the build task. Finally, we wait for a glance id for the built image to be returned which we set in the metadata imagefactory stores locally so that we can find the image and use it later.

There is more to the implementation of the Nova plugin, but that gets beyond the scope of this article. If you are interested in all of the gory details, have a look in the imagefactory git [repository][].

So, there you have it. Plugins for imagefactory. 


[repository]: https://github.com/redhat-imaging/imagefactory
[documentation]: http://imgfac.org/documentation/plugins.html
[OSDelegate]: https://github.com/redhat-imaging/imagefactory/blob/master/imgfac/OSDelegate.py
[CloudDelegate]: https://github.com/redhat-imaging/imagefactory/blob/master/imgfac/CloudDelegate.py
[Nova Image Builder]: https://github.com/redhat-imaging/novaimagebuilder