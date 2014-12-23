Lustre Hadoop Plugin
=======================

INTRODUCTION
------------

This document describes how to use Lustre (http://www.gluster.org/) as a backing store with Hadoop.

This plugin replaces the hadoop file system (typically, the Hadoop Distributed File System) with the 
LustreFileSystem, which writes to a shared Lustre mount point that is accessible by all machines
in the Hadoop cluster.

** NOTE: This plugin is based off of the GlusterFS Hadoop plugin with proprietary Seagate
enhancements for Lustre. The original plugin, with source, is available
[here](https://forge.gluster.org/hadoop) with some documentation
[here](https://forge.gluster.org/hadoop/pages/Architecture). **


REQUIREMENTS
------------

  * Supported OS is GNU/Linux
  * Lustre client installed on all machines in the cluster
  * Java Runtime Environment (JRE, for Hadoop)

NOTE: Plugin relies on one *nix command line utility to function properly:

  * getfattr: Used to fetch Extended Attributes of a file

Make sure this is installed on all hosts in the cluster and their locations are in $PATH
environment variable.


INSTALLATION
------------

Building the plugin from source requires [Maven](http://maven.apache.org/) and the JDK.

Change to glusterfs-hadoop directory in the GlusterFS source tree and build the plugin.

  * cd lustre_hadoop/
  * mvn package -DskipTests

On a successful build the plugin will be present in the `target` directory.
(NOTE: version number will be a part of the plugin)

Copy the plugin to lib/ directory in your $HADOOP_HOME dir.

  * cp target/lustrefs-0.0.1.jar $HADOOP_HOME/lib

Reference the example core-site.xml file to configure your Hadoop to use the plugin.

CONFIGURATION
-------------

  All plugin configuration is done in a single XML file (core-site.xml) with <name><value> tags in each <property>
  block.

  Brief explanation of the tunables and the values they accept (change them where-ever needed) are mentioned below

```
  name:  fs.lustrefs.impl
  value: org.apache.hadoop.fs.lustrefs.LustreFileSystem

         The default FileSystem API to use (there is little reason to modify this).

  name:  fs.default.name
  value: lustrefs:///

         The default name that hadoop uses to represent file as a URI (typically a server:port tuple). Use any host
         in the cluster as the server and any port number. This option has to be in server:port format for hadoop
         to create file URI; but is not used by plugin.

  name:  fs.lustrefs.mount
  value: /

         This is the directory where Lustre is mounted

```

USAGE
-----

  * Mount Lustre on all client nodes to the same mount point (e.g. /mnt/lustre).
  * Simply start Hadoop (mapred or yarn, in particular) and all cluster I/O should go to Lustre.


FOR HACKERS
-----------

Source Layout (./src/) followed by the usual painful Java hierarchy.

For the overall architecture, see.  Currently, we use the hadoop RawLocalFileSystem as 
the basis - and wrap it with the LustreVolume class.  That class is then used by the 
Hadoop 1x (LustreFileSystem) and Hadoop 2x (LustreFs) adapters.

 * ./conf/core-site.xml, sample configuration file
 * ./pom.xml, Maven build file
 * ./COPYING, Apache license for the original source
 * ./README.md, this file


