# hadoop-setup
Hadoop setup in AWS EC2 instances.


Now is the world of Big Data. Every company by now has collected tons of data since the dawn of cheap hard disks. However now comes the age to analyse it and make it useful for lots of business cases. The problem and the most critical challenge is to analyse such huge sets of data within a short period of time. To achieve this particaular goal, HADOOP was developed. 

The most comsuming time is to read the data serially. To solve this issue, engineers came up with the idea of reading data in parallel using distributed file systems and then doing batch analysis. Hence Hadoop came into existence. Hadoop has four main components.

Distributed file system (way to store the data in different storage devices)
MapReduce (Map: takes input from different data sources as <key,value> pairs and produces another set of intermediate <key,value> pairs as output after some processes. Reduce: uses the output from the mappers and aggregates them for that key.)
Hadoop Common (tools to read data from the hadoop file system)
YARN (resource manager for data storage and analysis)
Hadoop can easily be used by a readymade service called AWS EMR. However it can be costlier than setting up using just EC2 because EMR charges you not only for EMR service but also in addition for the EC2 instances being used for the EMR. That being said EMR is a better option if time is the constraint rather than money. In this blog we will focus on the case where money is the main issue and hence will show how to setup hadoop using just multiple EC2 instances in AWS or using any mutiple instances.

This article is based on Ubuntu (18.04.3 LTS).

Instance setup
Launch an instance and install the following dependencies.

$sudo apt-get update -y

$sudo apt-get upgrade -y

$sudo apt-get install openjdk-11-jdk-headless -y

$wget https://archive.apache.org/dist/hadoop/common/hadoop-3.3.0/hadoop-3.3.0.tar.gz

$sudo tar zxvf hadoop-* -C /usr/local

$sudo mv /usr/local/hadoop-* /usr/local/hadoop

(Note: Update the java and the hadoop url with the latest version)


Add the following environment variables.

# Add environment variables in bashrc
 
$vim ~/.bashrc

# Insert the following lines in bashrc file
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64

export PATH=$PATH:$JAVA_HOME/bin

export HADOOP_HOME=/usr/local/hadoop

export PATH=$PATH:$HADOOP_HOME/bin

export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop


# Update the current session with new environment variable.

$source ~/.bashrc

# Check if you can see the new variables.

$env

(Note: Change JAVA_HOME folder based on the java version installed.)

If you are using AWS, then create an image for this instance because we need to launch more instances with the same settings installed.

(if AWS is being used) Launch DataNode instances using the image and name them datanode1 and datanode2. Please note that these instances uses the same pem file for ssh access. Also enable public ips for every node (that will help to access datanodes from web UI).

We now need to make sure that namenode instance can communicate to the datanode instances over ssh without password. To achieve that we need to create a public key using ssh-keygen and then copy it to the ~/.ssh/authorized_keys file of the datanodes instance and as well as for the namenode as well.
Create a public key for ssh.

$ssh-keygen -f ~/.ssh/id_rsa -t rsa -P ""

$cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

Add the following to the .ssh/config file of namenode instance.


Host namenode

 HostName <public DNS of namenode instance>
 
 User ubuntu
 
Host datanode1
 
 HostName <public DNS of datanode1 instance>
 
 User ubuntu
 
Host datanode2
 
 HostName <public DNS of datanode2 instance>
 
 User ubuntu
 
 
Copy the public key (id_rsa.pub) to the ~/.ssh/authorized_keys file of datanode instances and then try to access the datanodes from namenode instance using ssh.

Configuration of the Hadoop cluster:-

Settings in both namenode and datanode instances.

$cd $HADOOP_CONF_DIR

Insert the following in core-site.xml file:

<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://<namenode public dns name>:9000</value>
  </property>
</configuration>


Note: Make sure the port 9000 is open in firewall because port 9000 is used for hdfs communication.
fs.defaultFS settings indicates where to communicate to the NameNode.


Insert the following in the yarn-site.xml file.

<configuration>

<!— Site specific YARN configuration properties -->

  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value><namenode public dns name></value>
  </property>
</configuration>
 

Insert the following in the mapred-site.xml.


<configuration>
  <property>
    <name>mapreduce.jobtracker.address</name>
    <value><namenode public dns name>:54311</value>
  </property>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>


Note: Make sure the port 54311 is open in firewall. mapreduce.jobtracker.address sets the node where the job tracker is running and the port being used for communication & mapreduce.framework.name sets MapReduce to run on YARN.


Settings in namenode instance only

Edit and add the following to /etc/hosts file

<namenode ip> <namenode_hostname>
<datanode1 ip> <datanode1_hostname>
<datanode2 ip> <datanode2_hostname>
127.0.0.1 localhost

Note: the hostname of the instance can be found using: $echo $(hostname). The IPs can be private IPs as well if they are in the same subnet.

Change to hadoop directory by $cd $HADOOP_CONF_DIR and add the following to hdfs-site.xml.


<configuration>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///usr/local/hadoop/data/hdfs/namenode</value>
  </property>
</configuration>

The data in each datanode is replicated to all datanodes and this acts as a failsafe option in cases where one of the node fails and stops responding. dfs.replication value sets the number of times data needs to be replicated across all datanodes. dfs.namenode.name.dir sets the location of keeping the namenode data. Since this location does not exist, we need to create one by:

$sudo mkdir -p $HADOOP_HOME/data/hdfs/namenode
 

Create a file called masters in HADOOP_CONF_DIR. This file will contain the hostname of master. This file sets the location of the Secondary namenode. In this case, it will be the namenode hostname.

<namenode_hostname>

Edit and add to the file called workers in HADOOP_CONF_DIR. This file will contain the hostnames of datanodes (they should be the same as the ones added in /etc/hosts file). 

<datanode1_hostname>
<datanode2_hostname>


Change the ownership of HADOOP_HOME to ubuntu.

$sudo chown -R ubuntu $HADOOP_HOME

Settings in datanode instances only
Add the following to $HADOOP_CONF_DIR/hdfs-site.xml file.

<configuration>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///usr/local/hadoop/data/hdfs/datanode</value>
  </property>
</configuration>
 

Run the following steps as it was done for namenode instance.

$sudo mkdir -p $HADOOP_HOME/data/hdfs/datanode
$sudo chown -R ubuntu $HADOOP_HOME
 

Initiating the Hadoop cluster
Go to namenode and run the following:

$hdfs namenode -format
$$HADOOP_HOME/sbin/start-dfs.sh

It will format the HDFS filesystem first and then initiate the HDFS.


Initiate YARN
$ $HADOOP_HOME/sbin/start-yarn.sh
 

If everything seems Ok, then check the daemon processes running.
$jps

To stop all Hadoop related processes.
 $ $HADOOP_HOME/sbin/stop-all.sh
 

To start all Hadoop related processes.
$ $HADOOP_HOME/sbin/start-all.sh
 

To get report of all the status of each slave

$hdfs dfsadmin -report
 

To refresh all the nodes
$hdfs dfsadmin -refreshNodes
 

Lastly, to view the namenode web UI, go to <namenode public DNS>:9870 and make sure the port is open in firewall settings.

Troubleshooting:
For 1GB RAM settings in the namenode yarn-site.xml should be:
  <property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>768</value>
  </property>
  <property>
    <name>yarn.scheduler.maximum-allocation-mb</name>
    <value>768</value>
  </property>
  <property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>64</value>
  </property>
  <property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
  </property>
 

Settings in the mapred-site.xaml should be:
  <property>
    <name>yarn.app.mapreduce.am.resource.mb</name>
    <value>256</value>
  </property>
  <property>
    <name>mapreduce.map.memory.mb</name>
    <value>128</value>
  </property>
  <property>
    <name>mapreduce.reduce.memory.mb</name>
    <value>128</value>
  </property>
 

Incompatible clusterID
$$HADOOP_HOME/sbin/stop-all.sh
$sudo rm -rf /tmp/hadoop-ubuntu
$sudo rm -rf $HADOOP_HOMEdata/hdfs/namenode   # directory mentioned in hdfs-site.xml for namenode
$sudo mkdir -p $HADOOP_HOME/data/hdfs/namenode
$sudo chown -R ubuntu $HADOOP_HOME
$hdfs namenode -format -force
$$HADOOP_HOME/sbin/start-all.sh
 

That's it. You have your Hadoop ready and running. More details can be found in the documentation. 
