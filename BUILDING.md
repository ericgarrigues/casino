# Building a Krakend docker image and deploying it in Kubernetes 

This chapter explains how to build and deploy krakend in a k8s cluster. 

In all the following examples you need to replace the **MY_REGISTRY** variable with the correct endpoint for your docker registry.

## Needed tools, applications and k8s cluster setup

The following software are needed :

 - git
 - docker
 - kubectl
 - krakend

On the Kubernetes side you must have a working load-balancer (often provided by the cloud stack) and an ingress controller.

## Configuration

With krakend, all API endpoints, backend definitions and behavior is controlled from a single file so it's easy to keep track of the changes in a version control system.

Also, a simple command can validate the configuration which will prevent putting in production a syntax invalid one so validation before deployment is easy.

```bash
krakend check --config configuration.json 
```

## Packaging of the application

I've decided to package the binary along with its configuration as a **docker image**.

When taking the binary directly from the docker hub registry, the needed docker file to build the image is pretty simple.

```
FROM devopsfaith/krakend
COPY configuration.json /etc/krakend/krakend.json
```

Then to build the docker image tagged with the corresponding version :

```
docker build -t krakend-apigw:v1 .
```

## Deployment in a docker registry
 
First we need to tag for deployment on the registry.

```
docker tag krakend-apigw:v1 MY_REGISTRY/krakend-apigw:v1
```

Then we'll push into it.

```
docker push MY_REGISTRY/krakend-apigw:v1
```

## Deploying in kubernetes

Before depploying the application in k8s, we need to define our replication policy to fulfill our 1000 req/s requirements and also to allow rollling update.

To install our docker image in k8s, we need to write a deployment configuration file :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: krakend
spec:
  selector:
    matchLabels:
      app: krakend
  replicas: 3
  template:
    metadata:
      labels:
        app: krakend
    spec:
      containers:
      - name: krakend
        image: MY_REGISTRY/krakend-apigw:v1
        ports:
        - containerPort: 8080
        command: [ "/usr/bin/krakend" ]
        args: [ "run", "-d", "-c", "/etc/krakend/krakend.json", "-p", "8080" ]
```

Then apply it to deploy a **pod** :

```bash
kubectl apply -f krakend-deployment.yaml
```

We also need to add a **service** in front of the pod to foward traffic to the containers :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: krakend-service
spec:
  type: NodePort
  ports:
  - name: http
    port: 8000
    targetPort: 8080
    protocol: TCP
  selector:
    app: krakend
```

Create service in the namespace :

```bash
kubectl apply -f krakend-service.yaml
```

And finally, we can add an **ingress** controller to expose and forward external traffic to the service.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: krakend-ingress
    annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
spec:
    rules:
    - http:
        paths:
        - path: /
          backend:
            serviceName: krakend-service
            servicePort: 8080
```

Command : 

```bash
kubectl apply -f krakend-ingress.yaml
```

Finally, i've choosen to have minimum 3 replicas and maximum 10 (containers) of the app and set the autoscaling policy based on the requests per seconds.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: krakend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: krakend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        name: krakend-ingress
      target:
        type: Value
        value: 1k
```

Again, apply it to deploy the HPA policy :

```bash
kubectl apply -f krakend-hpa.yaml
```

## Applying configuration changes to the gateway

To apply changes to the API gateway configuration, the easiest and recommanded way is to change the **configuration.yaml**, update the docker image, push to the registry the version with a new tag and initiate a rolling update of the krakend pod.

```bash
docker build -t krakend-apigw:v2 .
docker tag krakend-apigw:v2 MY_REGISTRY/krakend-apigw:v2
docker push MY_REGISTRY/krakend-apigw:v2
```

```bash
kubectl set image deployments/krakend krakend=MY_REGISTRY/krakend-apigw:v2
```

Verify that pods are restarted and updated :

```bash
kubectl get pods -l app=krakend
NAME                            READY   STATUS    RESTARTS   AGE
pod/krakend-7b8669fd46-db9zw    1/1     Running   0          31s
pod/krakend-7b8669fd46-664d5    1/1     Running   0          24s
pod/krakend-7b8669fd46-shzwf    1/1     Running   0          17s
```

```bash
kubectl describe pods/krakend-7b8669fd46-db9zw
Name:         krakend-7b8669fd46-db9zw
Namespace:    default
Priority:     0
Node:         k8s/10.81.74.65
Start Time:   Wed, 30 Sep 2020 18:15:14 +0200
Labels:       app=krakend
              pod-template-hash=7b8669fd46
Annotations:  cni.projectcalico.org/podIP: 10.1.77.25/32
              cni.projectcalico.org/podIPs: 10.1.77.25/32
Status:       Running
IP:           10.1.77.25
IPs:
  IP:           10.1.77.25
Controlled By:  ReplicaSet/krakend-7b8669fd46
Containers:
  krakend:
    Container ID:  containerd://2d9324f7e90d60ce5ae489135188033b7f2cbe40a2e723d35a59f1748cfe0806
    Image:         MY_REGISTRY/krakend-apigw:v2
    Image ID:      MY_REGISTRY/krakend-apigw@sha256:705f2f7a5cd550ec4e552846827ce24073a567d77a35c14a04206a554025d141
    Port:          8080/TCP
    Host Port:     0/TCP
    Command:
      /usr/bin/krakend
    Args:
      run
      -d
      -c
      /etc/krakend/krakend.json
      -p
      8080
    State:          Running
      Started:      Wed, 30 Sep 2020 23:28:52 +0200
    Last State:     Terminated
      Reason:       Unknown
      Exit Code:    255
      Started:      Wed, 30 Sep 2020 18:15:21 +0200
      Finished:     Wed, 30 Sep 2020 23:24:12 +0200
    Ready:          True
    Restart Count:  1
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-gb7cd (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-gb7cd:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-gb7cd
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:          <none>
```

