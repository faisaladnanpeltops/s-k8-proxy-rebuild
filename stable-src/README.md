# Deploy Reverse Proxy Setup on Kubernetes

## Install kubernetes

### Rancher

Rancher is a container management platform built for organizations that deploy containers in production. Rancher makes it easy to run Kubernetes everywhere, meet IT requirements, and empower DevOps teams.

### Installation steps to deploy K8s cluster on EC2 instances using Rancher

1. Deploy rancher server using docker

```
docker run -d --restart=unless-stopped \
  -p 8080:80 -p 8443:443 \
  --privileged \
  rancher/rancher:latest
```

Once the docker is running, it takes few minutes to initialize the server. Once the server is started, access the rancher UI on https://<host or IP>:8443

2. Setup AWS cloud credentials

Under profile, select "Cloud Credentials" and click on "Add Cloud Credentails". Populate the details of region, access key, secret key, credentails name and save it.

3. Create an ec2 node template.

Under profile, select "Node templates" and click on "Add template". Choose Amazon ec2 type for node template. 

Under Account Access, Choose the region where k8s cluster needs to be deployed. Choose the cloud credentails that is created in step 2.

Under Zone and Network, choose the Availability zone and the subnet where cluster nodes should be deployed. Then click next.

Under Secuirity groups, choose stander to automatically create a security group with required rules for cluster nodes. Then click next.

Under Instance, choose the instance type, root disk size etc.

Give a name for the node template and click on Create.


4. Create a K8s cluster.

Go to Clusters in rancher UI.

Click on Add cluster. Provide a cluster name and Name prefix for nodes.

Select the previously created template from step 3 in the dropdown and give the number of nodes required in the count field.

Select etcd, control plane and worker to make sure they are installed in at least 1 node.

Click on create button to provision the k8s cluster.


5. Test the cluster deployment:

Select and open the cluster to be tested. On the right top, click on "Kubeconfig File" and copy the config file data.

Create a local file called `kubeconfig` and paste the copied data.

Use this file to connect to the cluster by running below commands. Please note, in the below command the `KUBECONFIG` variable should be set to the path of `kubeconfig` file created in the previous step. It is easy to connect to the cluster, if the file is merged with `~/.kube/config` or if the file is placed in `stable-src` directory which will be created in the next step so that the file can be used to deploy the helm chart.

  ```
  export KUBECONFIG=kubeconfig
  kubectl get nodes
  kubectl get all --all-namespaces
  ``` 

## Apps deployment


### Clone the https://github.com/k8-proxy/k8-reverse-proxy repository and checkout `develop` branch

```
git clone https://github.com/k8-proxy/k8-reverse-proxy.git --recursive
cd stable-src
```

### Build and push docker images to a container registry. Below are example commands to show how to build and push docker images to docker registry.

Replace the <docker registry> with the docker registry or the dockerhub username in below commands

Login to the docker registry from the terminal before running below commands to push the images to docker registry.

Below is example command to login to a docker registry(dockerhub)

`docker login -u <username> -p <password>`

```
docker build nginx -t <docker registry>/reverse-proxy-nginx:0.0.1
docker push <docker registry>/reverse-proxy-nginx:0.0.1

docker build squid -t <docker registry>/reverse-proxy-squid:0.0.1
docker push <docker registry>/reverse-proxy-squid:0.0.1

wget -O c-icap/Glasswall-Rebuild-SDK-Evaluation/Linux/Library/libglasswall.classic.so https://github.com/filetrust/Glasswall-Rebuild-SDK-Evaluation/releases/download/1.117/libglasswall.classic.so # Get latest evaluation build of GW Rebuild engine
docker build c-icap -t <docker registry>/reverse-proxy-c-icap:0.0.1
docker push <docker registry>/reverse-proxy-c-icap:0.0.1
```

### Create and setup SSL certificates to use in proxy website.

Gather one or more DNS names for which SSL certificate is needed.

Example: 
    ```gov.uk.glasswall-icap.com,www.gov.uk.glasswall-icap.com,assets.publishing.service.gov.uk.glasswall-icap.com```

Update the DNS server(e.g. Route53) with all the DNS names pointing to the IP address of the ingress service in the kubernetes cluster.
For a single node cluster, This IP will be the IP address of the instance.

SSH into the instance and run below commands to generate certificate and private key. The last command will prompt for the list of DNS names for which this certificate will be used. Enter the DNS names in comma or space separated as shown above.

```
sudo apt-get update -y
sudo apt-get install certbot python3-certbot-nginx -y
sudo certbot certonly --nginx
```

Below files will be generated on the server. Please note that below file paths will vary based on the domain names passed.

Example:

```
/etc/letsencrypt/live/gov.uk.glasswall-icap.com/fullchain.pem
/etc/letsencrypt/live/gov.uk.glasswall-icap.com/privkey.pem
```

Generate base64 values of certificate and private key. Please note that below file paths will vary based on the domain names passed.

```
crt=$(sudo cat /etc/letsencrypt/live/gov.uk.glasswall-icap.com/fullchain.pem | base64)
key=$(sudo cat /etc/letsencrypt/live/gov.uk.glasswall-icap.com/privkey.pem | base64)
```

Pass above values in the helm deployment command in the next step. Please note the commands in next step might be executed in a different terminal, in this case, please use the values from above command in place of variables $crt and $key.


### Deploy to Kubernetes
Update the below command with the image name and image tag used in above step.

Image tag "0.0.1" in below command is just used as an example. Update below command with the same image tag used in above step(Build and push).

From `stable-src` directory of `s-k8-proxy-rebuild` repository run below commands to deploy the helm chart.

Make sure the variable `KUBECONFIG` is pointing to the path of `kubeconfig` file from the current terminal.

Please note if you want to use an external ICAP server, add these 2 lines in the middle of the below command after updating the IP address.

If ICAP_URL is set, icap server will not be deployed as part of this helm chart.

```
--set application.nginx.env.ICAP_URL=icap://<IP address>:1344/gw_rebuild \
--set application.squid.env.ICAP_URL=icap://<IP address>:1344/gw_rebuild \
```

```
helm upgrade --install \
--set image.nginx.repository=<docker registry>/reverse-proxy-nginx \
--set image.nginx.tag=0.0.1 \
--set image.squid.repository=<docker registry>/reverse-proxy-squid \
--set image.squid.tag=0.0.1 \
--set image.icap.repository=<docker registry>/reverse-proxy-c-icap \
--set image.icap.tag=0.0.1 \
--set ingress.tls.crt=$crt \
--set ingress.tls.key=$key \
reverse-proxy chart/
```

Verify that all pods(nginx, squid, icap) are running by executing below command
```
kubectl get pods
```

Once all the pods are running, there are 2 options to connect to the proxied website.

1. Forward the traffic from local machine to nginx service.

If the below command gives permission error to bind the port 443, please run the command with `root` user.

```
kubectl port-forward svc/reverse-proxy-reverse-proxy-nginx 443:443
```

You have to assign all proxied domains to the localhost address by adding them to hosts file ( `C:\Windows\System32\drivers\etc\hosts` on Windows , `/etc/hosts` on Linux )
  for example: 

```
127.0.0.1 gov.uk.glasswall-icap.com www.gov.uk.glasswall-icap.com assets.publishing.service.gov.uk.glasswall-icap.com
```

2. Connect using nginx ingress.

You have to assign all proxied domains to the IP address of the machine where helm chart is deployed by adding them to hosts file ( `C:\Windows\System32\drivers\etc\hosts` on Windows , `/etc/hosts` on Linux )
  
  For example: 

```
54.78.209.23 gov.uk.glasswall-icap.com www.gov.uk.glasswall-icap.com assets.publishing.service.gov.uk.glasswall-icap.com
```

You can test the stack functionality by downloading [a rich PDF file](https://assets.publishing.service.gov.uk.glasswall-icap.com/government/uploads/system/uploads/attachment_data/file/901225/uk-internal-market-white-paper.pdf) through the proxy and testing it against [file-drop.co.uk](https://file-drop.co.uk)
You can also visit [the proxied main page](https://www.gov.uk.glasswall-icap.com/) and make sure that the page loads correctly and no requests/links is pointing to the original `*.gov.uk` or other malformed domains.

### Customize deployment configuration
chart/values.yaml file can be updated to pass different environment variables, docker image repository and image tag to use.

for e.g. ALLOWED_DOMAINS, ROOT_DOMAIN, SUBFILTER_ENV etc. environment variable values can be updated to use a different domain.
