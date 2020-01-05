# 1. Create an influx db deployment and expose service in default namespace.
# 2. Create a deployment plan with below 2 containers – 
# a)	nginx
#       	Note: nginx should not start until a separate influxDB deployment is ready .
# b)	busybox that will capture the logs from the first container and write to a directory on the file system.
# 3. Create a service for the nginx.  
# 4. Create a new namespace and run a POD to get the IPs of services created above.

##Create below alias to be used

----------------------------------------------------------------------------
alias cl=clear
alias k=kubectl
alias kp='k get po -o wide'
----------------------------------------------------------------------------


1. nginx deployment file

----------------------------------------------------------------------------

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: sre
  name: sre
  namespace: default
spec:
 replicas: 1
 selector:
    matchLabels:
      name: sre
 template:
  metadata:
   labels:
    name: sre
  spec:
   volumes:
   - name: logs
     emptyDir: {}
   - name: backup
     hostPath:
      path: /var/log/nginx
      type: Directory
   initContainers:
      - name: influx-check
        image: bizongroup/alpine-curl-bash
        command: ["/bin/sh", "-c"]
        args:
         - until curl --max-time 1 influxdb:8086/ping;do echo 'influx not running. trying again after 2s'; sleep 2;done 
   containers:
   - name: nginx
     image: nginx
     volumeMounts:
     - name: logs
       mountPath: /var/log/nginx
   - name: busybox
     image: busybox
     volumeMounts:
     - name: logs
       mountPath: /var/log/nginx
     - name: backup
       mountPath: /var/log/bkp
     command: ["/bin/sh", "-c"]
     args:
      - while true; do
          cp -f /var/log/nginx/*.log /var/log/bkp/;
          sleep 10;
        done
----------------------------------------------------------------------------

## create directory in node where POD is to be scheduled
mkdir /var/log/nginx

## Deploy POD

## Expose service

k expose deploy sre --type ClusterIP --port 80

## Create influx db POD
----------------------------------------------------------------------------
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: influxdb
  name: influxdb
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: influxdb
  template:
    metadata:
      labels:
        run: influxdb
    spec:
      containers:
      - image: influxdb
        imagePullPolicy: Always
        name: influxdb
----------------------------------------------------------------------------

##Expose deployment 

kubectl expose deployment influxdb --type ClusterIP --port 8086

## Create NS -newns

kubectl create ns newns

## Get IP address of services from newns

k run -n newns --generator=run-pod/v1 alpine --rm --image bizongroup/alpine-curl-bash -it -- nslookup sre.default.svc.cluster.local|grep sre.default|awk '{print $3}'
k run -n newns --generator=run-pod/v1 alpine --rm --image bizongroup/alpine-curl-bash -it -- nslookup influxdb.default.svc.cluster.local|grep influxdb.default|awk '{print $3}'

