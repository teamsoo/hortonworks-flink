#### An Ambari Service for Flink
Ambari service for easily installing and managing Flink on HDP clusters.
Apache Flink is an open source platform for distributed stream and batch data processing
More details on Flink and how it is being used in the industry today available here: [http://flink-forward.org/?post_type=session](http://flink-forward.org/?post_type=session)


The Ambari service lets you easily install/compile Flink on HDP 2.4
- Features:
  - By default, downloads prebuilt package of Flink 1.2.1, but also gives option to build the latest Flink from source instead
  - Exposes flink-conf.yaml in Ambari UI

Limitations:
  - This is not an officially supported service and *is not meant to be deployed in production systems*. It is only meant for testing demo/purposes
  - It does not support Ambari/HDP upgrade process and will cause upgrade problems if not removed prior to upgrade
  - System-wide JAVA_HOME is needed in the target node

Author: [Ali Bajwa](https://github.com/abajwa-hw)
- Thanks to [Davide Vergari](https://github.com/dvergari) for enhancing to run in clustered env

#### Setup

- Download HDP 2.4 sandbox VM image or Azure setup from [Hortonworks website](http://hortonworks.com/products/hortonworks-sandbox/)
- Now start the VM
- After it boots up, find the IP address of the VM and add an entry into your machines hosts file. For example:
```
192.168.191.241 sandbox.hortonworks.com sandbox    
```
  - Note that you will need to replace the above with the IP for your own VM

- Connect to the VM via SSH (password hadoop)
```
ssh root@sandbox.hortonworks.com
```


- To download the Flink service folder, run below
```
VERSION=`hdp-select status hadoop-client | sed 's/hadoop-client - \([0-9]\.[0-9]\).*/\1/'`
sudo git clone https://github.com/teamsoo/hortonworks-flink.git   /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/FLINK   
```

- Restart Ambari
```
#sandbox
service ambari restart

#non sandbox
sudo service ambari-server restart
```

- Then you can click on 'Add Service' from the 'Actions' dropdown menu in the bottom left of the Ambari dashboard:

On bottom left -> Actions -> Add service -> check Flink server -> Next -> Next -> Change any config you like (e.g. install dir, memory sizes, num containers or values in flink-conf.yaml) -> Next -> Deploy

  - By default:
    - Container memory is 1024 MB
    - Job manager memory of 768 MB
    - Number of YARN container is 1

- On successful deployment you will see the Flink service as part of Ambari stack and will be able to start/stop the service from here:
![Image](../master/screenshots/Installed-service-stop.png?raw=true)

- You can see the parameters you configured under 'Configs' tab
![Image](../master/screenshots/Installed-service-config.png?raw=true)

- One benefit to wrapping the component in Ambari service is that you can now monitor/manage this service remotely via REST API
```
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
```

- ...and also install via Blueprint. See example [here](https://github.com/abajwa-hw/ambari-workshops/blob/master/blueprints-demo-security.md) on how to deploy custom services via Blueprints

#### Use Flink

- Run word count job
```
su flink
export HADOOP_CONF_DIR=/etc/hadoop/conf
cd /opt/flink
./bin/flink run --jobmanager yarn-cluster -yn 1 -ytm 768 -yjm 768 ./examples/batch/WordCount.jar
```
- This should generate a series of word counts
![Image](../master/screenshots/Flink-wordcount.png?raw=true)

- Open the [YARN ResourceManager UI](http://sandbox.hortonworks.com:8088/cluster). Notice Flink is running on YARN
![Image](../master/screenshots/YARN-UI.png?raw=true)

- Click the ApplicationMaster link to access Flink webUI
![Image](../master/screenshots/Flink-UI-1.png?raw=true)

- Use the History tab to review details of the job that ran:
![Image](../master/screenshots/Flink-UI-2.png?raw=true)

- View metrics in the Task Manager tab:
![Image](../master/screenshots/Flink-UI-3.png?raw=true)

#### Other things to try

- [Apache Zeppelin](https://zeppelin.incubator.apache.org/) now also supports Flink. You can also install it via [Zeppelin Ambari service](https://github.com/hortonworks-gallery/ambari-zeppelin-service) for vizualization

More details on Flink and how it is being used in the industry today available here: [http://flink-forward.org/?post_type=session](http://flink-forward.org/?post_type=session)


#### Remove service

- To remove the Flink service:
  - Stop the service via Ambari
  - Unregister the service

    ```
export SERVICE=FLINK
export PASSWORD=admin
export AMBARI_HOST=localhost
#detect name of cluster
output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://$AMBARI_HOST:8080/api/v1/clusters`
CLUSTER=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`

curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X DELETE http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE

#if above errors out, run below first to fully stop the service
#curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Stop $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "INSTALLED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE
    ```
   - Remove artifacts
    ```
    rm -rf /opt/flink*
    rm /tmp/flink.tgz
    ```   
