This document will assist you in getting a better understanding of how to use DOKS to deploy pods and deployments in DOKS and use Kubernetes services to access your deployments over the web.

First of all, we start with ways of accessing your DOKS cluster. you have two ways to access your Kubernetes cluster 

Using  kubeconfig

Using DOKS to access your cluster

Using Kubeconfig:

To connect using the kubeconfig. Download the kubeconfig from DigitalOcean control panel



Once you have the kubeconfig downloaded. You can use the --kubeconfig flag to access your cluster. for example: 

kubectl get pods -n <namespace> --kubeconfig <path to config file
kubectl get pods -n kubesystem --kubeconfig ~/Downloads/kubeconfig-level-stats-c3c14b4e-ee07-4b30-a552-6c02754e01c9

Using doctl

While connecting to the Kubernetes cluster using doctl. You’ll need to set the context for the cluster you want to access.

Use the below command to get the context 

doctl kubernetes cluster kubeconfig save <cluster-UUID>

Once the kubeconfig is saved, use the below command to list the available context

kubectl config get-contexts

You’ll see the below output showing the available context for your Kubernetes cluster

CURRENT   NAME                    CLUSTER                 AUTHINFO                      NAMESPACE

  *     do-blr1-testing         do-blr1-testing         do-blr1-testing-admin         

       do-sfo3-testing          do-sfo3-testing         do-sfo3-testing-admin   

Under current * shows the current context in use. You can use the below command to set the required context 

kubectl config use-context <context_name>

Now you’ll not need to add the --kubeconfig flag to access the cluster

Creating your first pod and deployment

Now as you have access to the cluster. Let’s start creating your first pod and deployment 

In Kubernetes, POD stands for "Process on a Deployment." It is the smallest and simplest unit in the Kubernetes object. You can simply create a pod on your Kubernetes cluster using the run command 

kubectl run <pod_name> --image <image_name> -n <namespace>

For example, you need to pod to be created with the Nginx image in a namespace called web

kubectl run nginx --image nginx -n web

After executing the above command you’ll see the below pod created:

 kubectl run nginx --image nginx -n web

pod/nginx created

kubectl get pod -n web

NAME    READY   STATUS    RESTARTS   AGE

nginx   1/1     Running   0          16s

You can also create a deployment to have multiple pods created in a single go. 

kubectl create deployment nginxdeploy --image nginx --replicas 3 -n web

Below is how you’ll see the deployment and pods after executing the above command

kubectl get deployment -n web

NAME          READY   UP-TO-DATE   AVAILABLE   AGE

nginxdeploy   3/3     3            3           12m

kubectl get pods -n web

NAME                          READY   STATUS    RESTARTS   AGE

nginx                         1/1     Running   0          50m

nginxdeploy-658b8cbc4-2jmg4   1/1     Running   0          13m

nginxdeploy-658b8cbc4-dhw6n   1/1     Running   0          13m

nginxdeploy-658b8cbc4-vmjk4   1/1     Running   0          13m

Container Registry

The DigitalOcean Container Registry (DOCR) is a private Docker image registry with additional tooling support that enables integration with your Docker environment and DigitalOcean Kubernetes clusters. 

So, now we have some clarity on how pods and deployments are created on your Kubernetes cluster. let me introduce you to using a Container Registry to push an image that you have created in-house.



Integrating the container registry can be done using a simple click from your DigitalOcean control panel. Please do not apart from the Container registry. You can integrate a Docker registry or another registry to push and pull images from your Kubernetes Cluster as well.

Integrating DOCR with Kubernetes cluster:

You can integrate your Kubernetes cluster with a container registry via the control panel. Just Navigate to Kubernetes >> Cluster name >> Settings >> DigitalOcean Container Registry Integration and click on edit.

Just check the container registry you have created and the registry should be Integrated. In case you want to use an API / CLI for integration. You can find more details here: https://docs.digitalocean.com/products/kubernetes/how-to/integrate-with-docr/ 



Pushing an image to DOCR:

For pushing an image for testing you can clone the test app by using the below command:

get clone https://github.com/rahul1269-kaushal/kubernetes_test_image.git ; cd kubernetes_test_image

Once you have cloned the repo. In the app folder, use the below command to build the image.

docker build -t <image_name>  .
or
docker build -t <image_name>:<tag_name>  .

<tag_name> is an optional input and by default, a tag name “latest” is added to the image if not specified. However, in a production environment is always a good practice to segregate the type of images you use 

After the Image build is successful you can check the image created using the docker images command. if successfully created you will see the below output:

root@droplet:~/kubernetes_test_image# docker images
REPOSITORY                                               TAG         IMAGE ID       CREATED         SIZE
test_image1                                              test        ebff25ddf542   5 minutes ago   85.6MB
test_image                                               latest      ebff25ddf542   5 minutes ago   85.6MB

Now we’ll be pushing this image to our container registry using DOCTL. 

Use the registry login command to integrate the docker with your container registry. You’ll be getting the below output on running the command:

root@droplet:~# doctl registry login

Logging Docker in to registry.digitalocean.com

Post logging in use the docker tag command to tag your image with the fully qualified destination path:

docker tag <my-image> registry.digitalocean.com/<my-registry>/<my-image>

in our case, we’ll be using the below command to push the test_image1 with the tag test we added

docker tag test_image1:test registry.digitalocean.com/testrahuldocker/test_image1:test

After tagging the image. It's time to push our image to the container registry using a docker push command. Make sure you use the full registry name in the format registry.digitalocean.com/<repo_name>/<image_name>

docker push registry.digitalocean.com/testrahuldocker/test_image1:test

Once the image is pushed successfully you’ll see the below output:

The push refers to repository [registry.digitalocean.com/testrahuldocker/test_image1]
011cf034b963: Layer already exists 
8f8c993167e2: Layer already exists 
34318ab8f942: Layer already exists 
46d998145bf9: Layer already exists 
edff9ff691d5: Layer already exists 
cbe4b9146f86: Layer already exists 
a6524c5b12a6: Layer already exists 
9a5d14f9f550: Layer already exists 
test: digest: sha256:e1ce8bcbbd09a34af709055afecd36e226852373e13fc94d6c8c63b512fb4245 size: 1994

You can also confirm if the push was successful by navigating to the container registry on the control panel and the Image name should be listed there.



Now you should be able to use your test_image to create your pods:

root@droplet:~# kubectl create deploy testapp --image registry.digitalocean.com/testrahuldocker/test_image1:test
deployment.apps/testapp created
root@reserve-ip-test:~# kubectl get po
NAME                                                      READY   STATUS      RESTARTS   AGE
testapp-64bbdc9477-cccf9                                  1/1     Running     0          8s

Kubernetes Service:

We are now familiar with creating pods and deployments using the publically available images or using your in-house image. However, we never tried accessing the application on these pods. If you try to access the node IP to access the app hosted on your pods. You’ll get no response. This is because we never exposed these pods.

This is when services come in. A Kubernetes service is a method for exposing a network application that is running as one or more pods in your cluster.

The two basic types of services you can use to expose your pods are NodePort and LoadBalancers.



NodePort:

Exposes the Service on each Node's IP at a static port (the NodePort). To make the node port available. One of the explicit ways to expose a deployment is using the expose command. Below is the basic syntax for using an expose command:

kubectl expose deployment <deployment-name> --type=NodePort --name=<service-name> --port=<port> --target-port=<target-port>

or if you want to do it implicitly you can use create a service YAML to deploy the service. Below is an example of a service YAML you can use to create:

# service.yaml

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: testapp
    name: testappexpose
spec:
  type: NodePort
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: testapp
  

You can then use the below command to create the service.yaml

kubectl create -f service.yaml

Both the above-mentioned ways will create a NodePort Service on your cluster to expose the application. Use the below command to check the created service.

root@rdroplet:~# kubectl get svc
NAME                                               TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
testappexpose                                      NodePort       10.245.29.196    <none>          8080:31487/TCP               14s

Now just fetch a node IP of your cluster and try accessing it over the node port. You should be able to access the service on your browser:



You might be wondering why the service was accessible on port 31487, this is because we never defined a node port of our choice. Node port is an optional parameter you can define this parameter in the yaml file. However, be sure it is within the node port range of 30000-32767.



Load Balancer:

Exposes the Service externally using an external load balancer. Kubernetes does not directly offer a load-balancing component; you must provide one, or you can integrate your Kubernetes cluster with a cloud provider.

The simplest way to create the DO load balancer is by creating a YAML config file 

#LB.yaml

apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/do-loadbalancer-healthcheck-check-interval-seconds: "3"
    service.beta.kubernetes.io/do-loadbalancer-healthcheck-healthy-threshold: "5"
    service.beta.kubernetes.io/do-loadbalancer-healthcheck-path: /
    service.beta.kubernetes.io/do-loadbalancer-healthcheck-port: "80"
    service.beta.kubernetes.io/do-loadbalancer-healthcheck-protocol: http
    service.beta.kubernetes.io/do-loadbalancer-healthcheck-response-timeout-seconds: "5"
    service.beta.kubernetes.io/do-loadbalancer-healthcheck-unhealthy-threshold: "3"
    service.beta.kubernetes.io/do-loadbalancer-size-unit: "3"
  name: todo-service
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    run: test
  type: LoadBalancer

Once you have the manifest ready. Use the create command to add the LB to your cluster. Once the LB is provisioned an external IP is assigned to the LB service. You can get the external IP by checking the created LB service

root@droplet:~# kubectl get svc
NAME                                               TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE

testappexpose                                      NodePort       10.245.29.196    <none>          8080:31487/TCP               19m
todo-service                                       LoadBalancer   10.245.164.36    146.190.9.25    80:31825/TCP               1m

You can now access the service hosted on your pod by using the external IP:
