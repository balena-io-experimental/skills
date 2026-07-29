---
title: "Use Out-of-Tree Linux Kernel Modules in balena"
source: "https://blog.balena.io/building-out-of-tree-linux-kernel-modules/"
author:
  - "[[balena]]"
published: 2023-06-30
created: 2026-03-25
description: "Learn how to use out-of-tree Linux kernel modules in your balenaCloud application."
---
BalenaOS is a lightweight, robust, secure, and reliable Linux-based embedded operating system designed to run containers that is available for hundreds of different device types. As is usual with Linux-based operating systems, most of the hardware drivers are contained in the operating system as kernel modules and are either automatically loaded by the operating system when required, or they can be specifically loaded by the application.

Mainline (in-tree) kernel modules are part of the Linux kernel source tree used by your device type and are considered part of the core Linux kernel. These modules are developed and maintained by the Linux kernel community. They undergo rigorous review processes and are subject to the kernel’s development and release cycles receiving ongoing maintenance and bug fixes from the kernel developers.

Devices that are not yet supported by the mainline kernel may use external third-party modules usually provided by the manufacturer. These out-of-tree modules are independent of the Linux kernel development process and are updated separately. They may lack the same level of review, testing, and maintenance as in-tree modules, potentially leading to compatibility issues or bugs and they usually require manual installation or configuration.

This post will explain how to build and load out-of-tree modules as part of your balenaCloud application.

## Building an out-of-tree kernel module

The standard process for building an out-of-tree kernel module is:

- Ensure all build dependencies are installed in your build host
- Install the kernel headers for your target kernel version – that is the version on the device that is going to load the module, not the kernel version on your build host
- Use the kernel build system to compile the module:  
	```
	cd /path/to/module/source
	make -C /path/to/headers modules_prepare
	make -C /path/to/headers M=$(PWD) modules
	```
	  
	The above assumes that the kernel module is built in the standard way, but your mileage may vary.
- The application must then load the kernel module when required

## BalenaOS kernel headers

BalenaOS provides kernel headers as a `kernel-modules-headers` compressed tarball artifact. Before using it, it must be built with `make modules_prepare` as seen above. Running `make modules_prepare` was optional in balenaOS 2.x but running `make modules_prepare` before use is mandatory in balenaOS 3.x.

BalenaOS 3.0 also removes the alternative `kernel-source` kernel headers artifact. Previous users of `kernel-source` must now modify their application to use `kernel-modules-headers` instead. This artifact did not contain full kernel sources for some time now despite its name – it became just kernel headers with balenaOS versions based on Yocto Thud and above, unfortunately without changing its name and without being removed from balenaOS.

## Building out-of-tree kernel modules as part of your containerized application

Balena provides a reference [kernel-modules-build](https://github.com/balena-os/kernel-module-build) project that deploys a multi-container application made up of two services:

- A `load` service that builds and loads an example out-of-tree kernel module using a multi-stage dockerfile
- A `check` service that depends on the `load` service above and verifies the module has been correctly loaded
```
version: '2'
services:
  load:
    build:
      context: ./module
      dockerfile: Dockerfile.template
    args:
      # Modify to the desired balenaOS version
      OS_VERSION: 2.108.27
      privileged: true
    restart: on-failure
  check:
    build:
      context: ./check
      dockerfile: Dockerfile.template
    depends_on:
        - load
```

To use it, you just need to modify the `OS_VERSION` argument in the `load` service above to match the balenaOS version of your target device.

Remember you can get a full list of available balenaOS versions using the [balenaCLI](https://github.com/balena-io/balena-cli/):

`balena os versions <deviceType>`

### Customizing with your own kernel source and application

In the example above, the `module/src` directory contains the example source for the `Hello World` module. You can replace this with the source code from your own kernel module.

Finally, the `check` service can also be replaced by your own application services.

An example of customizing the project for a custom out-of-tree kernel module can be found in the `alexgg/nvidia` branch, where the project is modified to use the [Nvidia GPU driver for x86 installer](https://www.nvidia.com/en-us/drivers/unix/).

## Conclusion

Building and loading out-of-tree kernel modules allows you to extend the capabilities of your IoT devices and tailor them to your specific requirements. By leveraging Balena’s multi-container approach, you can easily incorporate these modules into your applications and deploy them across your fleet of devices.

Remember to experiment, iterate, and contribute your knowledge back to the Balena community. Together, we can unlock the full potential of IoT devices and create innovative solutions that make a real impact.

As always, report any problem through our [support channels](https://www.balena.io/support/) or [forums](https://forums.balena.io/).

You can find out what other features we have in progress through our [public roadmap](https://roadmap.balena.io/). Let us know what you’d like to see us build and be sure to upvote features there.