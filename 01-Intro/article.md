# Introduction to Kubernetes
---

Kubernetes is a powerful evolution of docker-compose. It has a more verbose configuration but isn't that bad once you get a few key ideas.

## Prerequisites
- Docker desktop
- Enable Kubernetes in Docker desktop
- Kubectl
- Understands some fundamentals of containerization.

## Cool things you can do with Kubernetes
- Spin up more pods of your app easily.
- Automatically starts failing pods to keep your app running.
- Zero down-time deployments (rolling deploys).
- Load balancing across pods.
- Application Health/ready checks.

## What are words?
Image: A set of changes to build our app. Dockerfiles create Images. Each command in a dockerfile is it's own layer. Layers get cached and makes building dockerfiles much faster.

Container: A runtime instance of an Image. If Images are the blueprints a Container is the actual product.

Node: A Node is a machine capable of running containers. It can be a VM or a full server. A pod can only run on one node at a time. 

Pod: A pod is conceptually the same thing as a Container but for Kubernetes. Pods wrap a Container and add extra functionality like Liveliness checks and labeling. This is the smallest building block in Kubernetes. Pods are created and destroyed all the time.

Deployment: A deployment creates multiple pods. You normally only make pods through a deployment. Deployments are like the manager who restarts failing pods or updates them to new versions.

Service: A service is what hooks networks together. They are the glue that takes your external ip:port (localhost:3000) and forwards you to one of your pods ip:port (10.1.0.26:80).

## Kubectl basics
Kubectl is the command-line tool for Kubernetes. Here are some useful commands.
- `kubectl get [all|pods|deployments|services]`: displays stuff that is running. You can filter this to just Pods, Deployments, etc or use the `-l` flag to search by labels.
- `kubectl describe [name]`: Displays information about an item.
- `kubectl apply -f [filename.yaml]`: apply will create or update an item in Kubernetes. this is equivalent to `docker-compose up`.
- `kubectl delete [name]`: This will remove an item from Kubernetes.
- `kubectl port-forward [name] [port:port]`: If you don't have a service but want to access something the port-forward command will temporarily create a service to do so.

## Creating a pod
Lets start to get familiar with Kubernetes by making a pod. In practice you wouldn't make pods by hand but instead create a deployment. Deployments use the same syntax though so it will be good practice.

Create a new file. I normally give it the name followed by the type (`training.pod.yaml`). Yaml files can be either `.yml` or `.yaml` it doesn't matter.

Lets start adding some properties. All config files have an `apiVersion` and a `kind`.

``` yml
apiVersion: v1
kind: Pod
```

Next we can add some metadata about this pod. We are just going to add a name and a label. Labels allow you to group related configs together. You can also filter by labels.

``` yml
metadata:
    name: dummy-website
    labels:
        app: dummy-website
```

Finally lets add a container to our pod config. We will give it a name, set some resource limits, and hook up a port. I created a dummy nginx image for us to use.

``` yml
spec:
    containers:
    - name: dummy-website
      image: devync/dummy-website
      resources:
        limits:
            memory: "128Mi"
            cpu: "500m"
      ports:
        - containerPort: 80
```

If we want to deploy this pod we can run `kubectl apply -f ./training.pod.yaml`. To see if it actually worked run `kubectl get pods`. You should see `pod/dummy-website` and it should show 1/1 in the ready column. If you want to actually visit the site then run `kubectl port-forward pod/dummy-website 4210:80` then go to http://localhost:4210.

To keep things tidy for the next part lets delete that pod with `kubectl delete pod/dummy-website`.

## Create a deployment

Now that we understand pods lets put the pod we created in a deployment. Lets make a new file called `training.deployment.yaml`. The first part is going to look basically the same as our pod but this time the kind is a Deployment.

``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dummy-website
  labels:
    app: dummy-website
```

Next lets start filling out the spec. Here we can specify how many times we want our pod replicated. We also specify a selector which is used to tell this deployment what pods it will be in charge of. In our use case it might look redundant but in more complicated setups it can be useful.

``` yml
spec:
  replicas: 5
  selector:
    matchLabels:
      app: dummy-website
```
Lastly we need to add the template for the pod. This is actually exactly the same as our pod config we already wrote.

``` yml
  template:
    metadata:
      labels:
        app: dummy-website
    spec:
      containers:
      - name: dummy-website
        image: devync/dummy-website
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
```

To run this we do the same as we did with the pod and run `kubectl apply -f ./training.deployment.yaml`. We can do `Kubectl get all` to see everything that was created. You should now see that you have a deployment called dummy-website and 5 pods (not all of the pods might be running because we are running locally but you only really need one).

If you try to delete one of the pods and run get all again you will still see 5 pods. That is because the deployment saw that it was missing a pod and immediately started a new one to replace it. You can thing of it like these configs are the state we want the application to be in, and Kubernetes is trying to maintain that state.

You can run `kubectl port-forward [deployment name] 4210:80` and hit http://localhost:4210 again and see that our site works! It is pretty annoying to keep running this port-forward command though. Lets create a service so we don't have to.

## Create a service

Don't worry services are easy! Make a new file called `training.service.yaml`. There are several different types of services. The default is called ClusterIP which basically says let pods talk to me but nothing external, so you would still need to run the port-forward command. NodePort forwards a port from the actual Node machine to the Kubernetes network. This works but the port it gives you is kinda lame.

Now all documentation I can find says that LoadBalancer services shouldn't work locally, But they do! I think docker-desktop added an undocumented feature for them similar to minikube recently added support. Lets try it out and if not then we can change back to NodePort.

``` yml
apiVersion: v1
kind: Service
metadata:
  name: dummy-website
spec:
  selector:
    app: dummy-website
  type: LoadBalancer
  ports:
  - port: 4210
    targetPort: 80
```

This is super similar to what we have seen already. Some metadata, a kind, a selector like the deployment. We have the type that we talked about and we specify some ports. Apply that and then hit http://localhost:4210. If it doesn't work or if the External-IP of the service is stuck on pending then you'll have to switch back to NodePort.

## Conclusion
Kubernetes is super powerful, there are a lot more things we didn't cover like Volumes, Secrets, or ConfigMaps. Hopefully this makes Kubernetes seem less scary, It's not so bad once you use it a little bit.

---

## Resources

#### training.pod.yaml

``` yml
apiVersion: v1
kind: Pod
metadata:
  name: dummy-website
  labels:
    name: dummy-website
spec:
  containers:
  - name: dummy-website
    image: devync/dummy-website
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 80

```

#### training.deployment.yaml
``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dummy-website
  labels:
    app: dummy-website
spec:
  replicas: 5
  selector:
    matchLabels:
      app: dummy-website
  template:
    metadata:
      labels:
        app: dummy-website
    spec:
      containers:
      - name: dummy-website
        image: devync/dummy-website
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80

```

#### training.service.yaml
``` yml
apiVersion: v1
kind: Service
metadata:
  name: dummy-website
spec:
  selector:
    app: dummy-website
  type: LoadBalancer
  ports:
  - port: 4200
    targetPort: 80

```