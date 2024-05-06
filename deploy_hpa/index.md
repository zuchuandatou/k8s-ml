## Deploy a service with dynamic scaling. 


To start this deployment, we will run:

``` bash
// Get resources
git clone --single-branch --branch gh-pages https://github.com/teaching-on-testbeds/k8s-ml.git
sudo apt-get update; sudo apt-get -y install siege
sudo apt update; sudo apt -y install python3-pip # install pip if not already installed
python3 -m pip install kubernetes
wget https://file.io/XXXXXXXXXXXX -O k8s-ml/app/model.keras
ls -l k8s-ml/app/model.keras

// Build docker
docker build -t ml-app:0.0.1 ~/k8s-ml/app
docker tag ml-app:0.0.1  node-0:5000/ml-app:0.0.1
docker push node-0:5000/ml-app:0.0.1
echo http://$(curl -s ifconfig.me/ip):32000
sudo systemctl stop firewalld.service
docker run -p 32000:5000 node-0:5000/ml-app:0.0.1
docker ps -f ancestor=node-0:5000/ml-app:0.0.1 // crtl+c & check no longer running

// Start deployment
kubectl apply -f ~/k8s-ml/deploy_hpa/deployment_hpa.yaml
```

Let's check the status of the service. Run the command:

```bash
kubectl get svc -o wide
```

and then

```bash
kubectl get pods -o wide
```

Initially, you will see one pod in the deployment. Wait until the pod is "Running" and has a "1/1" in the "Ready" column.

Get the URL of the service - run

```bash
echo http://$(curl -s ifconfig.me/ip):32000
```

copy and paste this URL into your browser's address bar, and verify that your app is up and running there. 

### Test deployment under load

You will need two SSH sessions on node-0. In one, run

```bash
kubectl get hpa --watch
```

to see the current state of the autoscaler.


 In the second SSH session, run


```bash
siege -c 10 -t 360s http://$(curl -s ifconfig.me/ip):32000/test
screen -d -m python3 /home/cc/resource_monitor.py  -d 2400 -o /home/cc/resource_usage.csv
screen -d -m bash /home/cc/load_test.sh

tail load_output.csv
tail resource_usage.csv
```

Note that this test is of a longer duration, so that you will have time to observe additional replicas being brought up and becoming ready to use. 

### Stop the deployment

When you are done with your experiment, make sure to delete the deployment and services. To delete, run the command:

```bash
kubectl delete -f  ~/k8s-ml/deploy_hpa/deployment_hpa.yaml
```

Use

``` bash
kubectl get pods -o wide
```

and verify that (eventually) no pods are running your app.

Also, re-enable the firewall:

```bash
sudo systemctl start firewalld.service
```

