OpenTSDB/MapR-DB installation script
====================================

This project is a post install script for OpenTSDB to set it up to use MapR-DB instead of HBase.

##Step 0
Make sure you have either [mapr-client](http://doc.mapr.com/display/MapR/Installing+MapR+Software) or [mapr-core](http://doc.mapr.com/display/MapR/Installing+MapR+Software) installed

Make sure you have atleast Java 7 and GNUPlot installed for OpenTSDB to function
```sh
sudo apt-get install gnuplot
sudo apt-get install openjdk-7-jdk
```

##Step 1
####Option 1 - Install with a package manager
Download the appropriate installation package from https://github.com/OpenTSDB/opentsdb/releases
```sh
debian package
	sudo dpkg -i DOWNLOADED_FILE_PATH

rpm
	sudo rpm -i DOWNLOADED_FILE_PATH
```

####Option 2 - Install from source
Download the sources and execute the build script
```sh
git clone https://github.com/OpenTSDB/opentsdb.git
cd opentsdb
./build.sh
```

Install OpenTSDB
```sh
cd build
sudo make install
sudo mkdir /usr/local/share/opentsdb/plugins
sudo ln -s /usr/local/share/opentsdb/etc/opentsdb /etc/opentsdb
sudo ln -s /usr/local/share/opentsdb/ /usr/share/opentsdb
```

##Step 2
Specify the user that runs OpenTSDB:
- On debian based systems modify /etc/init.d/opentsdb and modify TSD_USER and TSD_GROUP
- On rpm based systems modify /etc/sysconfig/opentsdb and add USER=<Your user of choice>

Verify that /var/log/opentsdb is owned by the user that will be running the server (e.g. opentsdb user) and has 0755 permissions

##Step 3
You can set the path for the OpenTSDB tables to anything you desire. Just ensure that the base folder (e.g. /user/mapr) exists in your MapR Distributed File System. This same path will be used in __Step 6__.

Open /etc/opentsdb/opentsdb.conf, and edit the following properties

__NOTE:__ Be cautious to NOT have spaces at the end of your lines because the OpenTSDB properties parser may ignore your value if you have a space at the end of the line 
```ini
# ONLY set this to false for MapR-DB lower than v4.x
tsd.storage.enable_compaction = false

# ONLY set this to true for testing purposes, NEVER in production
tsd.core.auto_create_metrics = true

tsd.storage.hbase.data_table = /user/mapr/tsdb
tsd.storage.hbase.uid_table = /user/mapr/tsdb-uid
tsd.storage.hbase.meta_table = /user/mapr/tsdb-meta
tsd.storage.hbase.tree_table = /user/mapr/tsdb-tree

# MapR-DB does not utilize this value, but it must be set to something
tsd.storage.hbase.zk_quorum = localhost:5181
```
##Step 4
Get this post install setup project
```sh
git clone https://github.com/mapr-demos/opentsdb-maprdb-install.git
cd opentsdb-maprdb-install
```

##Step 5
Edit install.sh and set __HADOOP_HOME__ and __OPENTSDB_HOME__ to the correct folders if they do not match your setup *opentsdb default on debian is /usr/share/opentsdb*

Run this install script to download and copy all the necessary JAR files to opentsdb lib folder
```sh
sudo ./install.sh
```

##Step 6
Edit the create_tables.sh and set your TABLES_PATH to the same path you used in __Step 3__

Run the create tables script to create the required OpenTSDB tables in MapR-DB; The user that runs this script will be the default user with permissions for the table (e.g. you may want to run this as the mapr user). 

*This script sets permissions such that any user can read, write or append to the OpenTSDB tables. If you desire different permissions modify the create tables scripts as desired or use the [MapR Control System](http://doc.mapr.com/display/MapR/MapR+Control+System) to modify the permissions after running the script.*
```sh
su mapr -c ./create_table.sh
```

##Step 7
You can now validate the installation. The test data is in a time window starting 2014/08/04-10:16:00 and ending 2014/08/04-10:18:00 *adjust based on your timezone*
```sh
tsdb import test_data --auto-metric
tsdb scan --import 1y-ago sum mymetric.stock
```
Simply call ```service opentsdb start``` to start the tsd and ```service opentsdb stop``` to gracefully shutdown.
If you are new to OpenTSDB take a look at the [Quick Start guide](http://opentsdb.net/docs/build/html/user_guide/quickstart.html).
