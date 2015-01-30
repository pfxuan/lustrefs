#Lustre Hadoop Plugin

##INTRODUCTION

This document describes how to use Lustre as a backing store with Hadoop.

This plugin replaces the hadoop file system (typically, the Hadoop Distributed File System) with the 
LustreFileSystem, which writes to a shared Lustre mount point that is accessible by all machines
in the Hadoop cluster.

**NOTE: This plugin is based off of the GlusterFS Hadoop plugin with proprietary Seagate
enhancements for Lustre. The original plugin, with source, is available
[here](https://forge.gluster.org/hadoop) with some documentation
[here](https://forge.gluster.org/hadoop/pages/Architecture)**


##REQUIREMENTS

  * Supported OS is GNU/Linux
  * Lustre client installed on all machines in the cluster
  * Java Runtime Environment (JRE, for Hadoop)

NOTE: Plugin relies on one Unix command line utility to function properly:

  * getfattr: Used to fetch Extended Attributes of a file

Make sure this is installed on all hosts in the cluster and their locations are in $PATH
environment variable.


##INSTALLATION

Building the plugin from source requires [Maven](http://maven.apache.org/) and the JDK.

Change to lustre-hadoop directory and build the plugin.

  * cd lustre-hadoop/
  * mvn package -DskipTests

On a successful build the plugin will be present in the `target` directory. (NOTE: version number will be a part of the plugin and may be different than below). Copy the plugin to lib/ directory in your $HADOOP_HOME dir.

  * cp target/lustrefs-0.0.1.jar $HADOOP_HOME/lib
  
For example, with Cloudera, below is where the plugin jar can be placed on all nodes. For other distributions of Hadoop the plugin
jar should be placed in a similar location.

```
/opt/cloudera/parcels/CDH/lib/hadoop/lib/lustrefs-0.0.1.jar
```

##CONFIGURATION

Most plugin configuration is done in XML files with <name><value> tags in each <property> block. Brief explanation of the tunables and the values they accept (change them where-ever needed) are mentioned below. Note that Hadoop version 1 configuration values are not yet in this document.
  
###Required Changes

####core-site.xml

```
<property>
  <name>fs.lustrefs.impl</name>
  <value>org.apache.hadoop.fs.lustrefs.LustreFileSystem</value>
  <description>The FileSystem API to use.</description>
</property>

<property>
  <name>fs.AbstractFileSystem.lustrefs.impl</name>
  <value>org.apache.hadoop.fs.local.LustreFs</value>
  <description>The file implementation to use,
  in conjunction with the above.
  </description>
</property>

<property>
  <name>fs.default.name</name>
  <value>lustrefs:///</value>
  <description>The default name that hadoop uses to represent file as a URI.</description>
</property>

<property>
  <name>fs.lustrefs.mount</name>
  <value>/path/to/lustre</value>
  <description>This is the directory where Lustre is mounted</description>
</property>

```

#### mapred-site.xml

```
<property>
  <name>yarn.app.mapreduce.am.staging-dir</name>
  <value>/file/path/on/lustre</value>
  <description>Note that this path *should not* include the full path to lustre
  (i.e. skip the content of fs.lustrefs.mount).
  </description>
</property>
```

### Ecosystem changes

Several Hadoop ecosystem projects need special consideration to use this plugin.

#### Pig (pig.properties)

```
# Note that this path *should not* include the full path to lustre,
# similar to yarn.app.mapreduce.am.staging-dir.
pig.tmp.dir=/file/path/on/lustre

```

#### Hive (hive-site.xml)

```
<property>
  <name>hive.metastore.warehouse.dir</name>
  <value>/file/path/on/lustre</value>
  <description>Note that this path *should not* include the full path to lustre,
  similar to yarn.app.mapreduce.am.staging-dir.
  </description>
</property>

<property>
  <name>hive.exec.scratchdir</name>
  <value>/file/path/on/lustre</value>
  <description>Note that this path *should not* include the full path to lustre,
  similar to yarn.app.mapreduce.am.staging-dir.
  </description>
</property>
```


##USAGE

Using the plugin is straightforward, but with some file permission caveats. For all cases, these two steps are required:

  * Mount Lustre on all client nodes to the same mount point (e.g. /mnt/lustre).
  * Simply start Hadoop (mapred or yarn, in particular) and all cluster I/O should go to Lustre by default.

###FILE PERMISSIONS

The detais of running a Hadoop job using the Lustre plugin are different than when a job is run using HDFS.
This has to do with the fact that file permissions when using Lustre are handled through the usual POSIX paradigm,
instead of HDFS and its own file permissions system.
For example, when a MapReduce job is run using YARN in a Cloudera cluster,
without special configuration (see below), the Map and Reduce jobs are run as the 'yarn' user.
This means that the input, output, and staging directories for the job need to be accessible to the 'yarn' user.
This is obviously not acceptable for a multi-tenant system.

###SINGLE-TENANT CLOUDERA SETUP

If it is acceptible to have all the files needed for Hadoop jobs owned by/accessible to 'yarn,'
here are the steps required to make the system work.

  * The 'yarn' user needs to exist on all the Hadoop nodes (typically handled already when Cloudera is installed).
    Additionally, a 'yarn' user (with the same UID/GID as on the Hadoop nodes) needs to be created on the Lustre MGS/MDS.
    Lustre may have to be re-mounted after updating the users on the MGS/MDS.
  * Under the MapReduce staging dir (see `yarn.app.mapreduce.am.staging-dir` above), a 'yarn' directory owned by the
    user 'yarn' should be created. The directory can be automatically created if the staging dir has permissive enough
    permissions, but that is not recommended, in general.
  * Any input and output directories targeted by the job should have full permissions to the 'yarn' user.

###MULTI-TENANT CLOUDERA SETUP

**NOTE: this has not been tested by us yet! This is speculation.**
Hadoop has a special configuration called
["Secure Mode"](https://hadoop.apache.org/docs/r2.6.0/hadoop-project-dist/hadoop-common/SecureMode.html)
that allows for jobs to be run using the identity of the submitter, and not only 'yarn' (or other Hadoop-specific identity).
The Cloudera Manager has
[some ability to help set up](http://www.cloudera.com/content/cloudera/en/documentation/core/latest/topics/security.html)
Secure Mode.

The user permissions/identities on the Lustre system will also have to be set up to mirror the Hadoop nodes
using LDAP or similar.

While setting up Secure Mode will always be outside the purview of this README, we hope to include any tips or tricks
we learn if or when we investigate this ourselves.


##FOR HACKERS

Currently, we use the hadoop RawLocalFileSystem as 
the basis - and wrap it with the LustreVolume class.  That class is then used by the 
Hadoop 1x (LustreFileSystem) and Hadoop 2x (LustreFs) adapters.

 * src/, source files within the usual painfully deep Java hierarchy
 * conf/, sample configuration files
 * pom.xml, Maven build file
 * COPYING, Apache license for the original source, see glusterfs-hadoop.7z
 * README.md, this file
 * glusterfs-hadoop.7z, the original glusterfs-hadoop source this plugin is based on

##CONTACT

Stephen Skory <stephen.skory@seagate.com>

##COPYRIGHT

All modifications of this plugin beyond what was in the original GlusterFS-Hadoop source is copyright Seagate Technology.

