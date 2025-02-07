# ambari-presto-service

This repository contains the code and configuration needed to integrate [Presto](https://prestodb.io/) with [Ambari](https://ambari.apache.org/). Adding the Presto service to Ambari allows:

1. Installing and deploying Presto on a cluster from the Ambari UI.
2. Changing Presto configuration options via the Ambari UI.

# Table of contents

* [Getting Started](#getting-started)
  * [Requirements for integration](#requirements-for-integration)
  * [Adding the Presto service](#adding-the-presto-service)
  * [Supported topologies](#supported-topologies)
    * [Pseudo-distributed](#pseudo-distributed)
    * [Distributed](#distributed)
  * [Configuring Presto](#configuring-presto)
    * [Adding and removing connectors](#adding-and-removing-connectors)
* [Known issues](#known-issues)
* [Getting help](#getting-help)
* [Developers](#developers)
  * [Requirements for development](#requirements-for-development)
  * [Definitions](#definitions)
  * [Information on integrating services with Ambari](#information-on-integrating-services-with-ambari)
  * [Build and custom distributions](#build-and-custom-distributions)

# Getting Started

## Requirements for integration

1. Red Hat Enterprise Linux 6.x (64-bit) or CentOS equivalent.
2. You must have Ambari installed and thus transitively fulfill [Ambari's requirements](http://docs.hortonworks.com/HDPDocuments/Ambari-2.1.2.1/bk_Installing_HDP_AMB/content/_meet_minimum_system_requirements.html).
3. Oracle Java JDK 1.8 (64-bit). Note that when installing Ambari you will be prompted to pick a JDK. You can tell Ambari to download Oracle JDK 1.8 or point it to an existing installation. Presto picks up whatever JDK Ambari was installed with so it is imperative that Ambari is running on Oracle JDK 1.8.
4. Disable `requiretty`. On RHEL 6.x this can be done by editing the `/etc/sudoers` file and commenting out `Defaults    requiretty`.
5. Install `wget` on all nodes that will run a Presto component.

## Adding the Presto service

This section and all others that follow within [Getting Started](#getting-started) walk you through the integration steps needed to get Presto working with Ambari. By default, this integration code installs Presto version `0.130`, the latest version at the time of writing. To install the latest Teradata Presto release (0.127t), download the Ambari integration package from [here](http://it.teradata.com/PrestoDownload/?LangType=1040&LangSelect=true) and follow the remaining instructions below. To change the distribution to install another version, see [Build and custom distributions](#build-and-custom-distributions).

To integrate the Presto service with Ambari, follow the steps outlines below:

* Assuming HDP 2.3 was installed with Ambari, create the following directory on the node where the `ambari-server` is running:
```bash
$ mkdir /var/lib/ambari-server/resources/stacks/HDP/2.3/services/PRESTO
$ cd /var/lib/ambari-server/resources/stacks/HDP/2.3/services/PRESTO
```
* Place the integration files within the newly created PRESTO directory. Download the integration package that installs Teradata's '0.127t' from [here](http://it.teradata.com/PrestoDownload/?LangType=1040&LangSelect=true) or download the integration package that installs '0.130' from the releases section of this project. Upload the integration archive to your cluster and extract it like so:
```bash
$ tar -xvf /path/to/integration/package/ambari-presto-1.0.tar.gz -C /var/lib/ambari-server/resources/stacks/HDP/2.3/services/PRESTO
$ mv /var/lib/ambari-server/resources/stacks/HDP/2.3/services/PRESTO/ambari-presto-1.0/* /var/lib/ambari-server/resources/stacks/HDP/2.3/services/PRESTO
$ rm -rf /var/lib/ambari-server/resources/stacks/HDP/2.3/services/PRESTO/ambari-presto-1.0
```
* Finally, make all integration files executable and restart the Ambari server:
```bash
$ chmod -R +x /var/lib/ambari-server/resources/stacks/HDP/2.3/services/PRESTO/*
$ ambari-server restart
```
* Once the server has restarted, point your browser to it and on the main Ambari Web UI page click the `Add Service` button and follow the on screen wizard to add Presto. The following sections provide more details on the options and choices you will make when adding Presto.

## Supported topologies

The following two screens will allow you to assign the Presto processes among the nodes in your cluster. Once you pick a topology for Presto and finish the installation process it is impossible to modify that topology.

Presto is composed of a coordinator and worker processes. The same code runs all nodes because the same Presto server RPM is installed for both workers and coordinator. It is the configuration on each node that determines how a particular node will behave. Presto can run in pseudo-distributed mode, where a single Presto process on one node acts as both coordinator and worker, or in distributed mode, where the Presto coordinator runs on one node and the Presto workers run on other nodes.

The client component of Presto is the `presto-cli` executable JAR. You should place it on all nodes where you expect to access the Presto server via this command line utility. The `presto-cli` executable JAR does not need to be co-located with either a worker or a coordinator, it can be installed on its own. Once installed, the CLI can be found at `/usr/lib/presto/bin/presto-cli`.

*Do not place a worker on the same node as a coordinator.* Such an attempt will fail the installation because the integration software will attempt to install the RPM twice. In order to schedule work on the Presto coordinator, effectively turning the process into a dual worker/coordinator, please enable the `node-scheduler.include-coordinator` toggle available in the configuration screen.

### Pseudo-distributed

Pick a node for the Presto coordinator and *do not assign any Presto workers*. On the configuration screen that follows, you must also enable `node-scheduler.include-coordinator` by clicking the toggle.

### Distributed

Pick a node for the Presto coordinator and assign as many Presto workers to nodes as you'd like. Feel free to also place the client component on any node. Remember to not place a worker on the same node as a coordinator.

## Configuring Presto

The one configuration property that does not have a default and requires input is `discovery.uri`. The expected value is `http://<FQDN-of-node-hosting-coordinator>:8081`. Note that it is http and not https and that the port is 8081. If you change the value of `http-server.http.port`, make sure to also change it in `disovery.uri`.

Some of the most popular properties are displayed in the Settings tab (open by default). In the Advanced tab, set custom properties by opening up the correct drop down and specifying a key and a value. Note that specifying a property that Presto does not recognize will cause the installation to finish with errors as some or all servers fail to start.

Change the Presto configuration after installation by selecting the Presto service followed by the Configs tab. After changing a configuration option, make sure to restart Presto for the changes to take effect.

If you are running a version of Ambari that is older than 2.1 (version less than 2.1), then you must omit the memory suffix (GB) when setting the following memory related configurations: 'query.max-memory-per-node' and 'query.max-memory'. For these two properties the memory suffix is automatically added by the integration software. For all other memory related configurations that you add as custom properties, you'll have to include the memory suffix when specifying the value.

### Adding and removing connectors

To add a connector modify the `connectors.to.add` property, whose format is the following: `'{'connector1': ['key1=value1', 'key2=value2', etc.], 'connector2': ['key3=value3', 'key4=value4'], etc.}'`. Note the single quotes around the whole property and around each individual element. This property only adds connectors and will not delete connectors. Thus, if you add connector1, save the configuration, restart Presto, then specify {} for this property, connector1 will not be deleted.

To delete a connector modify the `connectors.to.delete` property, whose format is the following: `'['connector1', 'connector2', etc.]'`. Again, note the single quotes around the whole property and around each element. The above value will delete connectors `connector1` and `connector2`. Note that the `tpch` connector cannot be deleted because it is used to smoketest Presto after it starts. The impact that the presence of the `tpch` connector has on the system is negligible.

# Known issues

* For some older versions of Presto, when attempting to `CREATE TABLE` or `CREATE TABLE AS` using the Hive connector, you may run into the following error:
```
Query 20151120_203243_00003_68gdx failed: java.security.AccessControlException: Permission denied: user=hive, access=WRITE, inode="/apps/hive/warehouse/nation":hdfs:hdfs:drwxr-xr-x
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.check(FSPermissionChecker.java:319)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:219)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:190)
	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkPermission(FSDirectory.java:1771)
```
To work around the issue, edit your `jvm.config` settings by adding the following property `-DHADOOP_USER_NAME=hive`. This problem affects Presto `0.115t` but does not affect `0.127t`. After saving your edit to `jvm.config`, don't forget to restart all Presto components in order for the changes to take effect.

* If you decide to deploy an older version of Presto, you may have to adjust some setting manually. Please see [Configuring Presto](#configuring-presto) for an explanation of how to add custom settings. For example, the `task.max-memory` setting was deprecated in `0.127t` but is valid in `0.115t`. Therefore, if you're installing `0.115t` and would like to change `task.max-memory` to something other than its default, add it as a custom property.

* On the Presto service home page, if you click on 'Presto workers', you will get an incorrect list of workers. This is a known issue and has been fixed in Ambari 2.2.0.

* If the installation of Presto fails with `Python script has been killed due to timeout after waiting 1800 secs`, then the `wget` for either the Presto RPM or `presto-cli` JAR has timed out. To increase the timeout, increase the `agent.package.install.task.timeout` setting in `/etc/ambari-server/conf/ambari.properties` on the Ambari server host. Make sure to restart the Ambari server for the change to take effect. To resume, either hit the Retry button in the installation wizard, or finish the wizard and then install all Presto components individually by navigating to the relevant host and selecting Re-install. The components can be installed manually in any order but when starting the components, make sure to start the Presto coordinator last. If the installation keeps timing out, we suggest downloading the RPM and JAR outside the installation process, uploading them somewhere on your network and editing `package/scripts/download.ini` with the new URLs.

* At the moment, upgrading Presto from Ambari is not possible. Even though Ambari provides the capability to upgrade software, we didn't get a chance to implement the integration. If many users request this feature (if you'd like to see this feature let us know by commenting on [this issue](https://github.com/prestodb/ambari-presto-service/issues/17)), we'll add it to the next release. To upgrade Presto without the native upgrade integration you have to manually uninstall Presto and then install the new version.

# Getting help

If you're having trouble with anything, please [file a new issue](https://github.com/prestodb/ambari-presto-service/issues) and we'll try our best to help you out.

# Developers

## Requirements for development

1. Python 2.6/2.7.
2. [pip](https://pip.pypa.io/en/stable/installing/).
3. [make](https://www.gnu.org/software/make/).
4. `pip install -r $(REPO_ROOT_DIR)/requirements.txt`.

## Definitions

The following definitions, taken from the [Apache Ambari wiki](https://cwiki.apache.org/confluence/display/AMBARI/Stacks+and+Services), are useful when talking about integration:

1. Stack - Defines a set of Services and where to obtain the software packages for those Services. A Stack can have one or more version, and each version can be active/inactive. For example, Stack = HDP-2.3.
2. Service - Defines the Components (MASTER, SLAVE, CLIENT) that make up the Service. For example, Service = HDFS.
3. Component - The individual Components that adhere to a certain defined lifecycle (start, stop, install, etc). For example, Service = HDFS has Components = NameNode (MASTER), Secondary NameNode (MASTER), DataNode (SLAVE) and HDFS Client (CLIENT).

## Information on integrating services with Ambari

For more information on developing service integration code for Ambari, the following resources might be helpful:
1. [Webcast](http://hortonworks.com/partners/learn/#ambari) with Hortonworks engineers about integrating with Ambari. Includes slides and recorded video/audio of the talk.
2. Lots of [integration examples](https://github.com/abajwa-hw/ambari-workshops).

## Build and custom distributions

The build system for this project is very simple; it uses Python's standard [distutils](https://docs.python.org/2/distutils/) module. Calls to the `setup.py` script are wrapped with a Makefile to make common operations simple. Execute `make dist` to build the distribution, `make test` to run the unit tests and `make help` to get more info on all the available targets.

By default, the integration code installs Presto version `0.130`. Change the version displayed by Ambari when adding the Presto service by specifying a value for the `VERSION` variable when building the distribution. For example, to display Presto version `0.124`, run `make dist VERSION=0.124`. To download a different RPM and CLI to match version `0.124`, edit the `package/scripts/download.ini` file with URLs for both.
