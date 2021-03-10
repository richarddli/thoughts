<!--
This is an article template you can use as a starting point when writing DigitalOcean procedural tutorials about software development. This template is great for Python, JavaScript, or other softwar development tutorials that readers would follow.  Once you've reviewed the template, delete the comments and begin writing your outline or article. You'll find some examples of our custom Markdown at the very bottom of the template.

As you write, refer to our style and formatting guidelines for more detailed explanations:

- [do.co/style](https://do.co/style)

Use our [Markdown previewer](https://www.digitalocean.com/community/markdown) to review your article's formatting.

Readers should be able to follow your tutorial from the beginning to the end on their own computer. Before submitting your article to the editorial team, please be sure to test your article from start to finish on it exactly as written. Cut and paste commands from the article into your terminal to make sure there aren't typos in the commands. If you find yourself executing a command that isn't in the article, incorporate it into the article to make sure the reader gets the exact same results. We will test your article and send it back to you if we run into technical problems, which significantly slows down the publication process.
-->


# Rapid Development on Kubernetes with Telepresence

### Introduction

Application developers building microservices on Kubernetes quickly encounter two major problems that slow them down.

* Slow feedback loops. Once a code change is made, it must be deployed to Kubernetes to be tested. This requires a container build, push to a container registry, and deployment to Kubernetes. This adds minutes to every code iteration.
* Insufficient memory and CPU locally. Developers attempt to speed up the feedback loop by running Kubernetes locally with minikube or equivalent. However, resource-hungry applications quickly exceed the compute and memory available locally.

In this guide, you will configure [Telepresence](https://www.getambassador.io/products/telepresence/), a Cloud-Native Computing Foundation project that [addresses these problems](https://www.getambassador.io/use-case/local-kubernetes-development/).

## Telepresence

Telepresence is a Cloud-Native Computing Foundation project for fast efficient development on Kubernetes. With Telepresence, you 1) run your service locally, while you run the rest of your application in the cloud (2, 3).

![Telepresence 2 architecture](https://imgur.com/0wkJiox)

Telepresence creates a bi-directional network connection between your Kubernetes cluster and local workstation. This way, the service you’re running locally can communicate with services in the cluster, and vice versa.

This approach brings the following benefits:

* Your development environment has infinite scale. Using the cloud, you can have unlimited compute and memory.
* You can use your favorite tools locally. Since your development environment is local, you can use your preferred IDE, debugger, or any other tool that you need.
* You get rapid feedback. There’s no need to do a container build, push to registry, and deployment with every code change.

## Prerequisites

To complete this tutorial, you will need:

* A Kubernetes cluster such as [Digital Ocean Kubernetes](https://www.digitalocean.com/products/kubernetes/)
* `kubectl` installed locally on a workstation running either Linux or Mac OS X
* A local development environment for Node.js. You can follow [How to Install Node.js and Create a Local Development Environment](https://www.digitalocean.com/community/tutorial_series/how-to-install-node-js-and-create-a-local-development-environment).

Make sure you can perform a `kubectl` operation such as `kubectl get services` before proceeding.

## Step 1 — Install Telepresence

We're first going to install Telepresence locally. Telepresence comes as a single binary.

For Mac OS X users:

```command
# 1. Download the latest binary (~60 MB):
sudo curl -fL https://app.getambassador.io/download/tel2/darwin/amd64/latest/telepresence -o /usr/local/bin/telepresence
 
# 2. Make the binary executable:
sudo chmod a+x /usr/local/bin/telepresence
```

If you're on Linux:

```command
# 1. Download the latest binary (~50 MB):
sudo curl -fL https://app.getambassador.io/download/tel2/linux/amd64/latest/telepresence -o /usr/local/bin/telepresence
 
# 2. Make the binary executable:
sudo chmod a+x /usr/local/bin/telepresence
```

## Step 2 - Verify successful installation.

Telepresence connects your local workstation to a remote Kubernetes cluster. We'll test the installation is working correctly. We'll first connect to the cluster:

```command
telepresence connect
```

You'll see the following output:

```
[secondary_label Output]

Launching Telepresence Daemon
...
Connected to context default (https://<cluster public IP>)
```

Now, verify that Telepresence is working properly by connecting to the Kubernetes API server:

```command
curl -ik https://kubernetes
```

You should see the following output:

```
[secondary_label Output]
  HTTP/1.1 401 Unauthorized
  Cache-Control: no-cache, private
  Content-Type: application/json
  Www-Authenticate: Basic realm="kubernetes-master"
  Date: Tue, 09 Feb 2021 23:21:51 GMT
  Content-Length: 165
  {
    "kind": "Status",
    "apiVersion": "v1",
    "metadata": {
    },
    "status": "Failure",
    "message": "Unauthorized",
    "reason": "Unauthorized",
    "code": 401
  }%
```

Congratulations! You're able to use `curl` locally on your workstation to connect to the remote Kubernetes cluster, as if the Kubernetes cluster were running on your laptop.

## Step 3 — Install a Sample Node.js Application

We'll now install a sample NodeJS application to use with Telepresence. This NodeJS application consists of multiple services that communicate with each other.

Start by installing a sample application that consists of multiple services:

```command
kubectl apply -f https://raw.githubusercontent.com/datawire/edgey-corp-nodejs/main/k8s-config/edgey-corp-web-app-no-mapping.yaml
```

```
[secondary_label Output]
$ kubectl apply -f https://raw.githubusercontent.com/datawire/edgey-corp-nodejs/main/k8s-config/edgey-corp-web-app-no-mapping.yaml

deployment.apps/dataprocessingservice created
service/dataprocessingservice created
```

Give your cluster a few moments to deploy the sample application. Use `kubectl get pods` to check the status of your pods:

```
[secondary_label Output]
$ kubectl get pods

  NAME                                         READY   STATUS    RESTARTS   AGE
  verylargedatastore-855c8b8789-z8nhs          1/1     Running   0          78s
  verylargejavaservice-7dfddbc95c-696br        1/1     Running   0          78s
  dataprocessingservice-5f6bfdcf7b-qvd27       1/1     Running   0          79s
```

Once all the pods are in a `Running` state, go to the frontend service in your browser at [http://verylargejavaservice.default.svc.cluster.local:8080](http://verylargejavaservice.default.svc.cluster.local:8080).

You should see the EdgyCorp WebApp with a green title and green pod in the diagram.

## Step 4 - Set up a Local Development Environment

We'll now download the repository containing the services' code and run the NodeJS servicelocally. This version of the code has the UI color set to blue instead of green.

Clone the web app’s GitHub repository:

```command
git clone https://github.com/datawire/edgey-corp-nodejs.git
```

```
[secondary_label Output]
$ git clone https://github.com/datawire/edgey-corp-nodejs.git

  Cloning into 'edgey-corp-nodejs'...
  remote: Enumerating objects: 441, done.
  ...
```

Change into the repo directory, then into DataProcessingService:

```command
cd edgey-corp-nodejs/DataProcessingService/
```

Install the dependencies and start the Node server. 

```command
npm install && npm start
```

```
[secondary_label Output]
$ npm install && npm start

  ...
  Welcome to the DataProcessingService!
  { _: [] }
  Server running on port 3000
```


In a **new terminal window**, `curl` the service running locally to confirm it’s set to strong:

```command
curl localhost:3000/color
```

```
[secondary_label Output]
$ curl localhost:3000/color

  "blue"
```

## Step 5 - Intercept All Traffic to the Service

We’ll now create an intercept. An intercept is a rule that tells Telepresence where to send traffic. In this example, we will send all traffic destined for the `DataProcessingService` to the version of the `DataProcessingService` running locally instead:

Start the intercept with the `intercept` command, setting the service name and port

```command
telepresence intercept dataprocessingservice --port 3000
```

```
[secondary_label Output]
$ telepresence intercept dataprocessingservice --port 3000

  Using deployment dataprocessingservice
  intercepted
      Intercept name: dataprocessingservice
      State         : ACTIVE
      Destination   : 127.0.0.1:3000
      Intercepting  : all TCP connections
```

Go to the frontend service again in your browser. Since the service is now intercepted it can be reached directly by its service name at [http://verylargejavaservice:8080](http://verylargejavaservice:8080). You will now see the blue elements in the app. The frontend’s request to `DataProcessingService` is being intercepted and rerouted to the Node server on your laptop!

## Step 6 - Make a Code Change
We’ve now set up a local development environment for the `DataProcessingService`, and we’ve created an intercept that sends traffic in the cluster to our local environment. We can now combine these two concepts to show how we can quickly make and test changes.

1. Open `edgey-corp-nodejs/DataProcessingService/app.js` in your editor and change line 6 from `blue` to `orange`. Save the file and the Node server will auto reload.

2. Now, visit [http://verylargejavaservice:8080](http://verylargejavaservice:8080) again in your browser. You will now see the orange elements in the application.

We’ve just shown how we can edit code locally, and immediately see these changes in the cluster. Normally, this process would require a container build, push to registry, and deploy. With Telepresence, these changes happen instantly.

## Conclusion

In this article, we walked through a workflow for rapidly developing services on Kubernetes using Telepresence. With Telepresence, you're able to use your favorite development workflow. For more tutorials and information about Telepresence, see the [Telepresence documentation](https://www.getambassador.io/docs/latest/telepresence/quick-start/).