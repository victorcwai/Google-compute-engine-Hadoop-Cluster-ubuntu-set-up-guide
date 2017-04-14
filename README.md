# Google-compute-engine-ubuntu-Hadoop-Cluster-set-up-guide
The bash script to set up a Hadoop Cluster on Google compute engine using ubuntu 14.04 is as follow. 

Didn't use the google cloud sdk. Everything is done inside the instances (accessed via ssh).

Create at least 1 master node (dlp.srv.world) and 1 slave node (node1).

Remember to allow HTTP traffic and open the ports for web access (tcp 50070 and 8088).
```diff
#do this on all nodes
sudo apt-get install -y default-jdk
sudo useradd -d /usr/hadoop hadoop 
sudo mkdir /usr/hadoop
sudo chmod 755 /usr/hadoop 
sudo chown hadoop:hadoop /usr/hadoop 
echo "hadoop:hadoop" | sudo chpasswd

#do this on all nodes, while login as hadoop
sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
sudo service sshd restart

#do this on master node
ssh-keygen
#^need to enter passphrase
#send key to ALL nodes included localhost
ssh-copy-id localhost
ssh-copy-id node1
#^need to enter 'yes'

#do this to all nodes, change PasswordAuthentication back to no
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
sudo service sshd restart

#download and install hadoop, while login as hadoop
curl -O http://ftp.jaist.ac.jp/pub/apache/hadoop/common/hadoop-2.7.1/hadoop-2.7.1.tar.gz 
tar zxvf hadoop-2.7.1.tar.gz -C /usr/hadoop --strip-components 1

vi /usr/hadoop/.bash_profile
#add follows to the end
export HADOOP_HOME=/usr/hadoop
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_YARN_HOME=$HADOOP_HOME
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin

source /usr/hadoop/.bash_profile 

#do this on master node, then ssh to ALL nodes
mkdir ~/datanode 
ssh node1 "mkdir ~/datanode" 

#do this on master node only
vi ~/etc/hadoop/hdfs-site.xml
# add into <configuration> - </configuration> section
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///usr/hadoop/datanode</value>
  </property>
</configuration>
#send to all slaves
scp ~/etc/hadoop/hdfs-site.xml node1:~/etc/hadoop/ 

#ALL the script below is done on master node only
vi ~/etc/hadoop/core-site.xml
# add into <configuration> - </configuration> section
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://dlp.srv.world:9000/</value>
  </property>
</configuration>
# send to all slaves
scp ~/etc/hadoop/core-site.xml node1:~/etc/hadoop/ 

#NEED TO CHANGE JAVA PATH in ~/etc/hadoop/hadoop-env.sh#
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
# send to all slaves
scp ~/etc/hadoop/hadoop-env.sh node1:~/etc/hadoop/ 

mkdir ~/namenode 

vi ~/etc/hadoop/hdfs-site.xml
# add into <configuration> - </configuration> section
<configuration>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///usr/hadoop/namenode</value>
  </property>
</configuration>

vi ~/etc/hadoop/mapred-site.xml
# create new
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>

vi ~/etc/hadoop/yarn-site.xml
# add into <configuration> - </configuration> section
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>dlp.srv.world</value>
  </property>
  <property>
    <name>yarn.nodemanager.hostname</name>
    <value>dlp.srv.world</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
</configuration>

vi ~/etc/hadoop/slaves
# add all nodes (remove localhost)
dlp.srv.world
node1

#finally, boot the cluster
hdfs namenode -format
start-dfs.sh
start-yarn.sh
#check if running propoerly
jps 
hdfs dfsadmin -report

#execute a sample program 
hdfs dfs -mkdir /test
hdfs dfs -copyFromLocal ~/NOTICE.txt /test
hdfs dfs -cat /test/NOTICE.txt 
hadoop jar ~/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.1.jar wordcount /test/NOTICE.txt /output01 
hdfs dfs -ls /output01 
hdfs dfs -cat /output01/part-r-00000 
```
To start/stop Hadoop (just run it on master):
```
start-dfs.sh
stop-dfs.sh 

start-yarn.sh
stop-yarn.sh
```
Change permission of folders to hadoop if scp says permission denied.

# Reference
https://www.server-world.info/en/note?os=CentOS_7&p=hadoop
