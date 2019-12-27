# Lab: Build a Continuous Deployment Pipeline with Jenkins and Google Kubernetes Engine

This was originally forked from this [repo](https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes), but has since been updated and corrected with current information and best practices. Also, we'll be deploying from dockerhub, but you can change this to your own private registry.

## Introduction
This guide will take you through the steps necessary to continuously deliver your software to end users by leveraging [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) and [Jenkins](https://jenkins.io) to orchestrate the software delivery pipeline.
If you are not familiar with basic Kubernetes concepts, have a look at [Kubernetes 101](http://kubernetes.io/docs/user-guide/walkthrough/).

In order to accomplish this goal you will use the following Jenkins plugins, which will be installed later with our custom [Helm chart](https://helm.sh/docs/intro/):
  - [Jenkins Kubernetes Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin) - start Jenkins build executor containers in the Kubernetes cluster when builds are requested, terminate those containers when builds complete, freeing resources up for the rest of the cluster
  - [Jenkins Pipelines](https://jenkins.io/solutions/pipeline/) - define our build pipeline declaratively and keep it checked into source code management alongside our application code
  - [Google Oauth Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Google+OAuth+Plugin) - allows you to add your google oauth credentials to jenkins

In order to deploy the application with [Kubernetes](http://kubernetes.io/) you will use the following resources:
  - [Deployments](http://kubernetes.io/docs/user-guide/deployments/) - replicates our application across our kubernetes nodes and allows us to do a controlled rolling update of our software across the fleet of application instances
  - [Services](http://kubernetes.io/docs/user-guide/services/) - load balancing and service discovery for our internal services
  - [Ingress](http://kubernetes.io/docs/user-guide/ingress/) - external load balancing and SSL termination for our external service
  - [Secrets](http://kubernetes.io/docs/user-guide/secrets/) - secure storage of non public configuration information, SSL certs specifically in our case

## Prerequisites
1. A Google Cloud Platform Account, with valid billing information. $398 CAD is granted to free trial users.
2. [Enable the Compute Engine, Container Engine, and Container Builder APIs](https://console.cloud.google.com/flows/enableapi?apiid=compute_component,container,cloudbuild.googleapis.com)

## Setting up your Google Cloud
In this section you will start your [Google Cloud Shell](https://cloud.google.com/cloud-shell/docs/) and clone the lab code repository to it.

1. Create a new Google Cloud Platform project: [https://console.developers.google.com/project](https://console.developers.google.com/project)

2. Click the Google Cloud Shell icon in the top-right and wait for your shell to open:

  ![](docs/img/cloud-shell.png)

  ![](docs/img/cloud-shell-prompt.png)

3. When the shell is open, set your default account, project and compute zone. For these labs, set the default zone to 'us-east1-d':

  ```shell
  $ gcloud init
  ```

  You may also need to set the correct name of your project. e.g.:
  ```
  $ gcloud config set project gke-labs-26244

4. Clone the lab repository in your cloud shell, then `cd` into that dir:

  ```shell
  $ git clone https://github.com/KelfeinX/gke-labs.git
  Cloning into 'gke-labs'...
  ...

  $ cd gke-labs
  ```

## Creating a Kubernetes Cluster
You'll use Google Container Engine to create and manage your Kubernetes cluster. Provision the cluster with `gcloud`:

```shell
    gcloud container clusters create jenkins-ci --machine-type n1-standard-2 --num-nodes 2 --zone us-east1-d --scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform"
```

Once that operation completes download the credentials for your cluster using the [gcloud CLI](https://cloud.google.com/sdk/):
```shell
$ gcloud container clusters get-credentials jenkins-ci --zone us-east1-d --project=$(gcloud config get-value project)
Fetching cluster endpoint and auth data.
kubeconfig entry generated for jenkins-ci.
```

Confirm that the cluster is running and `kubectl` is working by listing the cluster info:

```shell
$ gcloud container clusters list
NAME        LOCATION    MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION    NUM_NODES  STATUS
jenkins-cd  us-east1-d  1.13.11-gke.14  34.73.126.148  n1-standard-2  1.13.11-gke.14  2          RUNNING
$ kubectl cluster-info
Kubernetes master is running at https://34.73.126.148
GLBCDefaultBackend is running at https://34.73.126.148/api/v1/namespaces/kube-system/services/default-http-backend:ht
tp/proxy
Heapster is running at https://34.73.126.148/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://34.73.126.148/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://34.73.126.148/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

## Installing Helm

In this lab, you will use Helm to install Jenkins from the Charts repository. Helm is a package manager that makes it easy to configure and deploy Kubernetes applications.  Once you have Jenkins installed, you'll be able to set up your CI/CD pipleline.

1. Download and install the helm binary

    ```shell
    wget https://storage.googleapis.com/kubernetes-helm/helm-v2.14.1-linux-amd64.tar.gz
    ```

2. Unzip the file to your local system:

    ```shell
    tar zxfv helm-v2.14.1-linux-amd64.tar.gz && cp linux-amd64/helm .
    ```

3. Add yourself as a cluster administrator in the cluster's RBAC so that you can give Jenkins permissions in the cluster:
    
    ```shell
    kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
    ```

4. Grant Tiller, the server side of Helm, the cluster-admin role in your cluster:

    ```shell
    kubectl create serviceaccount tiller --namespace kube-system && kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
    ```

5. Initialize Helm. This ensures that the server side of Helm (Tiller) is properly installed in your cluster.

    ```shell
    helm init --service-account=tiller
    helm repo update
    ```

6. It may take a few minutes to be ready. Afterwards, ensure Helm is properly installed by running the following command:

    ```shell
    helm version
    Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
    Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
    ```

## Installing and Configuring Jenkins
You will use a custom [values file](https://github.com/kubernetes/helm/blob/master/docs/chart_template_guide/values_files.md) to add the GCP specific plugin necessary to use service account credentials to reach your Cloud Source Repository.

1. Use the Helm CLI to deploy the chart with your configuration set.

    ```shell
    helm install --name jenkins stable/jenkins -f charts/jenkins.yaml --wait
    ```

2. Once that command completes ensure the Jenkins pod goes to the `Running` state and the container is in the `READY` state:

    ```shell
    $ kubectl get pods
    NAME                          READY     STATUS    RESTARTS   AGE
    jenkins-7c786475dd-vbhg4   1/1       Running   0          1m
    ```
    
3. Configure the Jenkins service account to be able to deploy to the cluster. 

    ```shell
    $ kubectl create clusterrolebinding jenkins-deploy --clusterrole=cluster-admin --serviceaccount=default:jenkins
    clusterrolebinding.rbac.authorization.k8s.io/jenkins-deploy created
    ```

4. Run the following command to setup port forwarding to the Jenkins UI from the Cloud Shell

    ```shell
    export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=jenkins" -o jsonpath="{.items[0].metadata.name}") && kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
    ```

5. Now, check that the Jenkins Service was created properly:

    ```shell
    $ kubectl get svc
    NAME               CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
    jenkins            10.35.249.67   <none>        8080/TCP    1h
    jenkins-agent      10.35.248.1    <none>        50000/TCP   1h
    kubernetes         10.35.240.1    <none>        443/TCP     4d19h
    ```

We are using the [Kubernetes Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin) so that our builder nodes will be automatically launched as necessary when the Jenkins master requests them.
Upon completion of their work they will automatically be turned down and their resources added back to the clusters resource pool.

Notice that this service exposes ports `8080` and `50000` for any pods that match the `selector`. This will expose the Jenkins web UI and builder/agent registration ports within the Kubernetes cluster.
Additionally the `jenkins-ui` services is exposed using a ClusterIP so that it is not accessible from outside the cluster.

## Connecting to Jenkins

1. The Jenkins chart will automatically create an admin password for you. To retrieve it, run:

    ```shell
    printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
    ```

2. To get to the Jenkins user interface, click on the Web Preview button![](../docs/img/web-preview.png) in cloud shell, then click “Preview on port 8080”:

![](docs/img/preview-8080.png)

You should now be able to log in with username `admin` and your auto generated password.

![](docs/img/jenkins-login.png)

## Preparing your Repo

TODO: Fork a repo

## Setting up a Pipeline

TODO: create a personal access token in GitHub, for Jenkins.

## Extra credit: deploy a breaking change, then roll back
Make a breaking change to the `gceme` source, push it, and deploy it through the pipeline to production. Then pretend latency spiked after the deployment and you want to roll back. Do it! Faster!

Things to consider:

* What is the Docker image you want to deploy for roll back?
* How can you interact directly with the Kubernetes to trigger the deployment?
* Is SRE really what you want to do with your life?

## Clean up
Clean up is really easy, but also super important: if you don't follow these instructions, you will continue to be billed for the Google Container Engine cluster you created.

To clean up, navigate to the [Google Developers Console Project List](https://console.developers.google.com/project), choose the project you created for this lab, and delete it. That's it.
