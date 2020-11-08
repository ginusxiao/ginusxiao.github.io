# 官网
[官网](https://github.com/SUSE/lrbd/wiki)

Simplifies iSCSI management of Ceph RBD images.

The lrbd utility centrally stores the configuration in Ceph objects and executes the necessary rbd and targetcli commands to apply the stored configuration.

The tool depends on targetcli which depends on python-rtslib. The original purpose is to support the rbd backstore and is the default option. If your distribution is lacking kernel support or updated packages for targetcli and python-rtslib, the iblock backstore can be used for many configurations.

# 源码
[源码](https://github.com/swiftgist/lrbd)

