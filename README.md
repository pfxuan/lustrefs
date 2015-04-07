#LUSTRE HADOOP PLUGIN

##INTRODUCTION

This document describes how to use Lustre as the primary backing store with Hadoop.
This plugin replaces the default hadoop file system (typically, the Hadoop Distributed File System, HDFS) with the 
Lustre File System, which writes to a shared Lustre mount point that is accessible by all machines
in the Hadoop cluster.
Please note that HDFS is compatible with this plugin and can be kept running.
It does not need to be turned off, although it is optional.

**NOTE: This plugin is based off of the GlusterFS Hadoop plugin with
modifications for Lustre. The original plugin, with source, is available
[here](https://forge.gluster.org/hadoop) with some documentation
[here](https://forge.gluster.org/hadoop/pages/Architecture).
We have included the GlusterFS plugin source within the repository as well.**


##REQUIREMENTS

  * Supported OS is GNU/Linux
  * Lustre client installed on all machines in the cluster and Lustre mounted to the same mountpoint
    on all nodes (e.g. `/mnt/lustre`)
  * Java Runtime Environment (JRE, for Hadoop)
  * Hadoop

##INSTALLATION

The installation has the following steps:

  1. Copy the LustreFS jar plugin to all Hadoop nodes
  2. Edit the XML files for all services that will access data on Lustre
  3. Decide how file permissions will be handled by choosing between single- or multi-tenant Hadoop
  4. Restart all Hadoop services and run jobs

###1. JAR FILE

The repository includes a pre-built jar file named lustrefs-hadoop-(version.number).jar.
This jar file needs to be placed in the [CLASSPATH](https://docs.oracle.com/javase/tutorial/essential/environment/paths.html)
of any and all Hadoop services that are to use the LustreFS plugin, and on all nodes of the cluster, including the master node.

For example, within Cloudera, below is where the plugin jar can be placed on all nodes for MapReduce to use the plugin:

```
/opt/cloudera/parcels/CDH/lib/hadoop-mapreduce/lustrefs-hadoop-(version.number).jar
```

###2. CONFIGURATION

Most plugin configuration is done in XML files with <name><value> tags in each <property> block.
These changes will need to be made identically on all nodes and services restarted to take effect.
  
###REQUIRED CONFIGURATION CHANGES

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
  <name>fs.defaultFS</name>
  <value>lustrefs:///</value>
  <description>The default name that hadoop uses to represent file as a URI.</description>
</property>

<property>
  <name>fs.lustrefs.mount</name>
  <value>/path/to/lustre/hadoop</value>
  <description>This is the directory where Lustre is mounted</description>
</property>
```

####mapred-site.xml

```
<property>
  <name>yarn.app.mapreduce.am.staging-dir</name>
  <value>/file/path</value>
  <description>Note that this path *should not* include the full path to lustre
  (i.e. skip the content of fs.lustrefs.mount).
  </description>
</property>
```

####hdfs-site.xml

If HDFS is running it should be configured *not* to use Lustre.
See below for an example of how to run jobs between HDFS and Lustre.

```
<property>
  <name>fs.defaultFS</name>
  <value>hdfs://namenode:8020/</value>
  <description>HDFS 
</property>
```

###ECOSYSTEM CONFIGURATION CHANGES

Several Hadoop ecosystem projects need special consideration to use this plugin.

####Pig (pig.properties)

```
# Note that this path *should not* include the full path to lustre,
# similar to yarn.app.mapreduce.am.staging-dir.
pig.tmp.dir=/file/path/on/lustre

```

####Hive (hive-site.xml)

```
<property>
  <name>hive.metastore.warehouse.dir</name>
  <value>/file/path</value>
  <description>Note that this path *should not* include the full path to lustre,
  similar to yarn.app.mapreduce.am.staging-dir.
  </description>
</property>

<property>
  <name>hive.exec.scratchdir</name>
  <value>/file/path</value>
  <description>Note that this path *should not* include the full path to lustre,
  similar to yarn.app.mapreduce.am.staging-dir.
  </description>
</property>
```

###3. FILE PERMISSIONS

The detais of running a Hadoop job using the Lustre plugin are different than when a job is run using HDFS.
This has to do with the fact that file permissions when using Lustre are handled through the usual POSIX paradigm,
instead of HDFS and its own internal file permissions system.
For example, when a MapReduce job is run using YARN in a Cloudera cluster,
without special configuration (see below), the Map and Reduce jobs are run as the 'yarn' user.
This means that the input, output, and staging directories for the job need to be accessible to the 'yarn' user.
This is obviously not acceptable for a multi-tenant system.

###SINGLE-TENANT SETUP

If it is acceptible to have all the files needed for Hadoop jobs owned by/accessible to 'yarn,'
here are the recommended steps required to make the system work:

  * After making the XML changes above, and before (re)staring Hadoop, create a directory on
    Lustre that is world-writeable (e.g. `/mnt/lustre/hadoop`).
    This directory should be the
    same as the value of `fs.lustrefs.mount` in the XML files.
    Hadoop (and ecosystem services)
    will make properly-owned directories under this directory upon (re)start.
    Once the various services are known to be working, the permissions of this directory can
    be rolled back. The final ownership of the directory is not important.
  * All UNIX accounts used for Hadoop services (e.g. 'yarn' or 'mapred') need to be created on the Lustre MGS/MDS.
    Specifically, the UIDs/GIDs, and group memberships of the Hadoop UNIX service accounts,
    need to be mirrored on the Lustre MGS/MDS.
  * A directory for the 'yarn' user (named 'yarn') will have to be created and `chown`ed in the
    `{$fs.lustrefs.mount}/user` directory. 


###MULTI-TENANT SETUP

Hadoop has a special configuration called
[Secure Mode](https://hadoop.apache.org/docs/r2.6.0/hadoop-project-dist/hadoop-common/SecureMode.html)
that allows for jobs to be run using the identity of the submitter.
When secure-mode Hadoop is configured, a user obtains a Kerberos "ticket" and then submits a job.
With the ticket, jobs will be run under the users UNIX identity, which will allow Hadoop jobs to access data
on Lustre as the user.

Configuring secure-mode Hadoop is outside the scope of this documentation.
Here are some brief notes for consideration:

  * The steps required for single-tenant Hadoop (above) should be followed with the addition that all
    users of Hadoop will also need accounts on the Lustre MGS/MDS.
  * Secure-mode Hadoop requires Kerberos. A Kerberos server will have to be available, and all Hadoop nodes
    configured to use it.
  * Any user using a secure-mode Hadoop cluster will need an account on all Hadoop nodes, which is not
    how a typical Hadoop cluster is set up.
    Configuring the Hadoop nodes to use LDAP (or equivalent) for logins may be beneficial.
  * Each user will need a directory named `{$fs.lustrefs.mount}/user/{$username}` and owned by that user.
  * We recommend turning on secure-mode Hadoop before configuring the cluster to use Lustre.
    Verify that the cluster works in secure-mode before switching to Lustre.
  * The Cloudera Manager has
    [a Kerberos Authentication Wizard](https://www.cloudera.com/content/cloudera/en/documentation/core/latest/topics/cm_sg_intro_kerb.html)
    that helps with setting up secure-mode Hadoop.
  * Hortonworks has
    [instructions on setting up secure-mode](http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.2.0/HDP_Man_Install_v22/index.html#Item1.26.3.2)
    within their framework.

###4. RESTART AND RUNNING WITH LUSTRE

After completing the steps above restart the cluster.

Running jobs with Lustre is very similar to using HDFS.
Paths are either relative to a users home directory, or absolute.
The following commands are equivalent:

  * `$ hadoop jar application.jar appName inputDir ouputDir`
  * `$ hadoop jar application.jar appName {$fs.lustrefs.mount}/user/{$username}/inputDir {$fs.lustrefs.mount}/user/{$username}/outputDir`

Users can access data anywhere on Lustre that they have permissions to, even if they are outside of the
`{$fs.lustrefs.mount}/user/{$username}` hierarchy.

HDFS can be run on the Hadoop nodes even with LustreFS as the default file system.
Jobs can be run between HDFS and Lustre. The command below illustrates a job that
takes data in from HDFS and outputs to Lustre:

`$ hadoop jar application.jar appName hdfs://hdfs_namenode:8020/user/username/d1 d2`

##FOR HACKERS

Currently, we use the hadoop RawLocalFileSystem as 
the basis - and wrap it with the LustreVolume class.  That class is then used by the 
Hadoop 1x (LustreFileSystem) and Hadoop 2x (LustreFs) adapters.

 * src/, source files within the usual painfully deep Java hierarchy
 * conf/, sample configuration files
 * pom.xml, Maven build file
 * COPYING, Apache license
 * README.md, this file
 * glusterfs-hadoop.7z, the original glusterfs-hadoop source this plugin is based on

###BUILDING

Building the plugin from source requires [Maven](http://maven.apache.org/) and the JDK.

Change to lustre-hadoop directory and build the plugin.

  * cd lustre-hadoop/
  * mvn package -DskipTests

On a successful build the plugin will be present in the `target` directory. (NOTE: version number will be a part of the plugin and may be different than below). Copy the plugin to lib/ directory in your $HADOOP_HOME dir.

  * cp target/lustrefs-(version.number).jar $HADOOP_HOME/lib

##CONTACT

For general inquiries, please contact <hadoop.on.lustre@seagate.com>.
For techinical topics, please contact Stephen Skory <stephen.skory@seagate.com>.

##COPYRIGHT

This source code is being released under the Apache license.
Please see the COPYING file included in this repository.
This is the same license as the GlusterFS Hadoop plugin that this plugin is based off of.

