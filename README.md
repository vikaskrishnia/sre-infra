1. Create an influx db deployment and expose service in default namespace.
2. Create a deployment plan with below 2 containers – 
 a)	nginx
       	Note: nginx should not start until a separate influxDB deployment is ready .
 b)	busybox that will capture the logs from the first container and write to a directory on the file system.
3. Create a service for the nginx.  
4. Create a new namespace and run a POD to get the IPs of services created above.
