
   

注册 登录   文件  发布      
    
An Ambari Service for Flink
Ambari service for easily installing and managing Flink on HDP clusters.
Apache Flink is an open source platform for distributed stream and batch data processing
More details on Flink and how it is being used in the industry today available here: http://flink-forward.org/?post_type=session

The Ambari service lets you easily install/compile Flink, now it has been upgraed to support flink 1.9.1 on HDP 3.1.1 for Ambari 2.7.4 (Redhat/CentOS7.x).
- Features:
- By default, downloads prebuilt package of Flink 1.9.1, but also gives option to build the latest Flink from source instead
- Exposes flink-conf.yaml in Ambari UI

Limitations:
- This is not an officially supported service and is not meant to be deployed in production systems. It is only meant for testing demo/purposes
- It does not support Ambari/HDP upgrade process and will cause upgrade problems if not removed prior to upgrade

Authors:
- Thanks to the orginial author: Ali Bajwa
- Thanks to Davide Vergari for enhancing to run in clustered env
- Thanks to Ben Harris for updating libraries to work with HDP 2.5.3
- Thanks to Anand Subramanian for updating libraries to work with HDP 2.6.5 and flink version 1.8.1
- Thanks to bright for updating to support flink v1.9.1 on HDP v3.1.1 for Ambari v2.7.4 (Redhat/CentOS7.x)

Setup
To download the Flink service folder, run below
VERSION=`hdp-select status hadoop-client | sed 's/hadoop-client - \([0-9]\.[0-9]\).*/\1/'`
sudo git clone https://github.com/abajwa-hw/ambari-flink-service.git   /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/FLINK   
To download the flink-1.9.1-bin-scala_2.12.tgz from https://flink.apache.org/downloads.html
 https://www-us.apache.org/dist/flink/flink-1.9.1/flink-1.9.1-bin-scala_2.12.tgz
To compile flink-shaded-hadoop-2-uber jar file from source code
(In order to make sure flink-1.9.1 works correctly, you have to compile flink-shaded-hadoop-2-uber using flink-shaded-9.0 since you cannot find the flink-shaded-hadoop-2-uber jar file from internet for hadoop 3.x version now （2019.12.10）. )
 wget https://www.apache.org/dyn/closer.lua/flink/flink-shaded-9.0/flink-shaded-9.0-src.tgz
 tar xvfz flink-shaded-9.0-src.tgz
 or get the source from https://github.com/apache/flink-shaded.git
 git clone https://github.com/apache/flink-shaded.git
 git checkout release-7.0
 cd flink-shaded-9.0
 mvn clean install -DskipTest -Dhadoop.version=3.1.1
 ll flink-shaded-hadoop-2-uber/target/
 (flink-shaded-hadoop-2-uber-3.1.1-9.0.jar is in flink-shaded-hadoop-2-uber/target/ )
To speed up the installation and resue the files, you can put flink-1.9.1-bin-scala_2.12.tgz and flink-shaded-hadoop-2-uber-3.1.1-9.0.jar into local www server.
yum install httpd
mkdir -p  /var/www/html/hdp/services/flink
cp -fv flink-1.9.1-bin-scala_2.12.tgz /var/www/html/hdp/services/flink/
cp -fv flink-shaded-hadoop-2-uber-3.1.1-9.0.jar /var/www/html/hdp/services/flink/
systemctl start httpd
then you can get the url, then the two urls in configuration/flink-ambari-config.xml

(please replace your www server ip and port)
http://192.168.101.85:8181/hdp/services/flink/flink-1.9.1-bin-scala_2.12.tgz
http://192.168.101.85:8181/hdp/services/flink/flink-shaded-hadoop-2-uber-3.1.1-9.0.jar
Change or Confirm the links are correct in configuration/flink-ambari-config.xml
  <property>
    <name>flink_download_url</name>
    <value>http://192.168.101.85:8181/hdp/services/flink/flink-1.9.1-bin-scala_2.12.tgz</value>
    <description>Snapshot download location. Downloaded when setup_prebuilt is true</description>
  </property>
  <property>
    <name>flink_hadoop_shaded_jar</name>
    <value>http://192.168.101.85:8181/hdp/services/flink/flink-shaded-hadoop-2-uber-3.1.1-9.0.jar</value>   
    <description>Flink shaded hadoop jar download location. Downloaded when setup_prebuilt is true</description>
  </property>
Restart Ambari
ambari-server restart
Then you can click on 'Add Service' from the 'Actions' dropdown menu in the bottom left of the Ambari dashboard:
On bottom left -> Actions -> Add service -> check Flink server -> Next -> Next -> Change any config you like (e.g. install dir, memory sizes, num containers or values in flink-conf.yaml) -> Next -> Deploy

By default:

Container memory is 1024 MB
Job manager memory of 768 MB
Number of YARN container is 1

On successful deployment you will see the Flink service as part of Ambari stack and will be able to start/stop the service from here:
Installed-service-stop.png?raw=true未知大小

You can see the parameters you configured under 'Configs' tab
Installed-service-config.png?raw=true未知大小

One benefit to wrapping the component in Ambari service is that you can now monitor/manage this service remotely via REST API

export SERVICE=FLINK
export PASSWORD=admin
export AMBARI_HOST=localhost
#detect name of cluster
output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://$AMBARI_HOST:8080/api/v1/clusters`
CLUSTER=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`
#get service status
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X GET http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE
#start service
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE
#stop service
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Stop $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "INSTALLED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE
...and also install via Blueprint. See example here on how to deploy custom services via Blueprints
Use Flink
Run word count job
su flink
export HADOOP_CONF_DIR=/etc/hadoop/conf
export HADOOP_CLASSPATH=`hadoop classpath`
cd /opt/flink
./bin/flink run --jobmanager yarn-cluster -yn 1 -ytm 768 -yjm 768 ./examples/batch/WordCount.jar
This should generate a series of word counts
Flink-wordcount.png?raw=true未知大小

Open the YARN ResourceManager UI. Notice Flink is running on YARN
YARN-UI.png?raw=true未知大小

Click the ApplicationMaster link to access Flink webUI
Flink-UI-1.png?raw=true未知大小

Use the History tab to review details of the job that ran:
Flink-UI-2.png?raw=true未知大小

View metrics in the Task Manager tab:
Flink-UI-3.png?raw=true未知大小

Other things to try
Apache Zeppelin now also supports Flink. You can also install it via Zeppelin Ambari service for vizualization
More details on Flink and how it is being used in the industry today available here: http://flink-forward.org/?post_type=session

Remove service
To remove the Flink service:
Stop the service via Ambari
Unregister the service
export SERVICE=FLINK
export PASSWORD=admin
export AMBARI_HOST=localhost
# detect name of cluster
output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://$AMBARI_HOST:8080/api/v1/clusters`
CLUSTER=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X DELETE http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE
If above errors out, run below first to fully stop the service

curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Stop $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "INSTALLED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE
Remove artifacts
rm -rf /opt/flink*
rm /tmp/flink.tgz
FAQ
user and group are not created when installing
You need to add the display-name and value-attributes fields to your flink-env.xml file on Ambari 2.7.x.
Please refer to https://community.cloudera.com/t5/Support-Questions/Cannot-add-a-custom-service-through-Ambari-2-7-0/td-p/194331
Check /var/log/flink/flink-setup.log for details of instllation
Delete the cache files when you change fink.py or other instllation files
ll /var/lib/ambari-agent/cache/stacks/HDP/3.1/services/FLINK/package/scripts/flink.py
rm -rf /var/lib/ambari-agent/cache/stacks/HDP/3.1/services/FLINK
hadoop: command not found
Issue log:
2019-12-10 10:33:07,077 - Writing File['/opt/flink/conf/flink-conf.yaml'] because contents don't match
2019-12-10 10:33:07,079 - Execute['hadoop fs -mkdir -p /user/flink'] {'ignore_failures': True, 'user': 'hdfs'}
2019-12-10 10:33:07,230 - Skipping failure of Execute['hadoop fs -mkdir -p /user/flink'] due to ignore_failures. Failure reason: Execution of 'hadoop fs -mkdir -p /user/flink' returned 127. -bash: hadoop: command not found
2019-12-10 10:33:07,231 - Execute['hadoop fs -chown flink /user/flink'] {'user': 'hdfs'}
2019-12-10 10:33:07,366 - Skipping stack-select on FLINK because it does not exist in the stack-select package structure.
Solved:
ln  -s  /usr/hdp/3.1.4.0-315/hadoop/bin/hadoop /etc/alternatives/hadoop  
flink-1.9.1-bin-scala_2.12.tgz and flink-shaded-hadoop-2-uber-3.1.1-9.0.jar are cached in /tmp
if you have to update both files, you should delete them from /tmp folder.
ll /tmp/flink*
Clean and re-install
Stop and delete flink from Ambari web page first
rm -rfv /var/lib/ambari-agent/cache/stacks/HDP/3.1/services/FLINK
rm -rfv /tmp/flink-1.*
rm -rfv /tmp/flink-shaded-hadoop*
rm -rfv /var/lib/ambari-server/resources/stacks/HDP/3.1/services/FLINK
rm -rfv /opt/flink
ambari-server restart
VERSION=`hdp-select status hadoop-client | sed 's/hadoop-client - \([0-9]\.[0-9]\).*/\1/'`
sudo git clone https://github.com/abajwa-hw/ambari-flink-service.git   /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/FLINK   
ambari-server restart
