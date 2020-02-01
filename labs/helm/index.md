# Working with Helm

## Install Helm 

```
wget https://raw.githubusercontent.com/helm/helm/master/scripts/get -O - | bash
```

Now that Helm is installed we need to install the backend 

Create tiller service account
```
kubectl create serviceaccount tiller --namespace kube-system
```

Grant tiller cluster admin role 
```
kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

Initialize Helm to install tiller in your cluster 
```
helm init --service-account=tiller --upgrade
helm repo update
```

## Chart Operations for a demo Chart

**Step 1:** Download dependent charts

```
helm dependency update charts/counter
```

You will now see that the `charts` directory has a new file 

```
ls charts/counter/charts 
```

Output should be similar to:  
```
mariadb-4.4.2.tgz
```

**Step 2:** Now that we have downloaded the dependant chart, create a new release.

`helm install --name counter charts/counter`

**Step 3:** List the release

`helm ls`

**Step 4:** Try the release out. Find the IP address of the service

`kubectl get svc`

**NOTE: The application can take a few minutes to come online, if it doesn't load wait 5 minutes and try again**

Load the 'External-IP' in a browser

You can also test with curl
```
curl <External-IP>
```

**Step 5:** Upgrade application

Let's upgrade our application by deploying a new release.   
**NOTE:** There is a bug with mariadb chart when using randomly generated passwords. To overcome this
we need to get the current password and save it as a variable to use when upgrading our release. 

Get password
```
kubectl get secret counter-mariadb -o yaml
```

In the `yaml` output you will see two encoded passwords, `mariadb-root` and `mariadb-password`, we need the mariadb-password. 
Decode the hashed password
```
echo '<mariadb-password>' | base64 --decode
```

Create a variable from the output
```
export DB_PASSWORD=<password from above> 
```

Now we can upgrade the release
```
helm upgrade --set image.tag=v0.0.2,mariadb.db.password=$DB_PASSWORD counter charts/counter
```

**Step 6:** Confirm application is being upgraded
```
kubectl get pods 
```
You should see pods being terminated and created
**NOTE: This may take a little while to occur**

**Step 7:** Confirm updated application is available    
Either load in a browser 'External-IP' or use `curl` to hit the ELB

**Step 8:** Rollback a release   
Sometimes an update does not go as planned, in these situations you can rollback to an earlier revision.  
First run `helm history counter` to see the available revisions.     
You should see there are two revisions, let's rollback to revision 1. 
```
helm rollback counter 1
```

Now you will see the pods go into a `Terminating` and `ContainerCreating` status as the old `v2` pods are removed and the new `v1` pods are deployed. 

After a few minutes you should be able to run the `curl` command and see it is now displaying version 1.

**Step 9:** Delete a release, but not the history   

`helm delete counter`

Now if you run `helm history counter` you will see it still exists. If you want to completely remove a release and it's history you can run
```
helm delete counter --purge
```

## Override Helm chart values at deployment    
Helm allows values to bet set at deployment time which makes charts more portable and customizable.   
In this lab you are going to deploy a simple `todo` app and use Helm to override some default vaules. 

**Step 1:** Let's start by adding a new repository which has a chart for our simple todo app.    
```
helm repo add bitnami https://charts.bitnami.com
```

**Step 2:** Now run `update` to index the charts in that repository   
```
helm repo update 
```

**Step 3:** After this we can install our app.   
```
helm install bitnami/mean --name todo
```

In the output you will see: 
```
1. Get the URL of your MEAN app by running:

  kubectl port-forward --namespace default svc/mean-mean 80:80
  echo "MEAN app URL: http://127.0.0.1:80/"
```

**Step 4:** Access application over `ClusterIP`   
Now that you've deployed the app you can run `kubectl port-forward` from above output to create a tunnel from your local machine
to the cluster. This allows you to load the application on your local machine for testing without making it publicly available.

**Step 5:** Override `serviceType` to expose application publicly.   

You have 2 options at this point. We can delete and reinstall from the bitnami repo and override with LoadBalancer or we can fetch the chart and `upgrade`.

Let's upgrade our application so that it is available externally. 
```
helm upgrade todo --set service.type=LoadBalancer
```

**Step 6:** Confirm application is accessible.   
Run the ouput from the `upgrade` command to get the URL the application is running on and load in your browser. 

**Step 7:** Deploy a new release 
Now the app is available publicly let's deploy an updated release.   
```
helm upgrade mean --set service.type=LoadBalancer,image.tag=8.16.1-r33
```

**Step 8:** Confirm the new revision has been deployed
```
helm history todo 
``` 

## Create your own Helm chart   
In this lab you will be creating a simple Helm chart that deploys nginx by default, you will then update the values so that it deploys a demo application. 

**Step 1:** Create chart  
```
helm create mychart
```

Helm will create a new directory in your project called mychart with the structure shown below. 

Step 2:** Navigate our new chart (pun intended) to find out how it works.   
```
mychart/
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

### Templates
The most important piece of the puzzle is the templates/ directory. This is where Helm finds the YAML definitions for your Services, Deployments and other Kubernetes objects. If you already have definitions for your application, all you need to do is replace the generated YAML files for your own. What you end up with is a working chart that can be deployed using the helm install command.   

These files are not meant to be static, Helm runs each of them through a Go rendering engine and extends the template language adding a number of features for writing charts.

Open the `templates/service.yaml` file to see what this looks like:   
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
{{ include "mychart.labels" . | indent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: {{ include "mychart.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
```

This is a basic Service definition using templating. When deploying the chart, Helm generates a definition that looks more like a valid Service. We can see this in 2 ways. 

1. Do a `dry-run` of `helm install` and enable debugging   
```
helm install --dry-run --debug ./mychart
```

You should see something like this:   
```
...
# Source: mychart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: open-sponge-mychart
  labels:
    app.kubernetes.io/name: mychart
    helm.sh/chart: mychart-0.1.0
    app.kubernetes.io/instance: open-sponge
    app.kubernetes.io/version: "1.0"
    app.kubernetes.io/managed-by: Tiller
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: mychart
    app.kubernetes.io/instance: open-sponge
...
```

2. Newer versions of Helm support running `helm template` which will take the template files and convert them to working definition files.   
```
helm template mychart -x templates/service.yaml
```

### Values
The template in `service.yaml` makes use of the Helm-specific objects `.Chart` and `.Values.`. The former provides metadata about the chart to your definitions such as the name, or version. The latter `.Values` object is a key element of Helm charts, used to expose configuration that can be set at the time of deployment. The defaults for this object are defined in the `values.yaml` file. Try changing the default value for `port` and execute another dry-run, or `helm template -x...` you should find that the `port` in the Service changes.

If a user of your chart wanted to change the default configuration, they could provide overrides directly on the command-line:

```
helm install --dry-run --debug mychart --set service.port=8080
```

Now use `helm template` and pass the same override.

The service.yaml template also makes use of partials defined in _helpers.tpl, as well as functions like replace, the templating language is powerful and takes some time to get familiar with.  The [Helm documentation](https://helm.sh/docs/chart_template_guide/) is a great place to get started and learn more. 

## Documentation 
Another file you should be aware of is `NOTES.txt` in the `template` directory.  This is a templated, plaintext file that will get printed out after the chart is successfully deployed. In this file you can put helpful 'next steps` for your users to access and use your application. Because `NOTES.txt is run through a template engine, you can use templating inside of it to print out working commands for obtaining an IP address, URL, or getting a password from a Secret object.

**Step 3:** Deploying your first chart   
The chart you generated in the previous step is setup to run an NGINX server exposed via a Kubernetes Service. By default, the chart will create a `ClusterIP` type Service, so NGINX will only be exposed internally in the cluster. To access it externally, we’ll use the `LoadBalancer` type instead. We can also set the name of the Helm release so we can easily refer back to it. Let’s go ahead and deploy our NGINX chart using the helm install command:
```
helm install --name example mychart --set service.type=LoadBalancer
```
Helm will now display the resources created as well as the contents of `NOTES.txt`   
```
NAME:   example
LAST DEPLOYED: Thu Sep 19 08:29:35 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME             READY  UP-TO-DATE  AVAILABLE  AGE
example-mychart  0/1    1           0          0s

==> v1/Pod(related)
NAME                              READY  STATUS             RESTARTS  AGE
example-mychart-57b6b6698b-vbqjl  0/1    ContainerCreating  0         0s

==> v1/Service
NAME             TYPE          CLUSTER-IP    EXTERNAL-IP  PORT(S)         AGE
example-mychart  LoadBalancer  10.100.60.13  <pending>    8080:32392/TCP  1s


NOTES:
1. Get the application URL by running these commands:
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace default svc -w example-mychart'
  export SERVICE_IP=$(kubectl get svc --namespace default example-mychart -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP:8080
```

Run the commands in the output to get a URL to access the NGINX service and pull it up in your browser.

If all went well, you should see the NGINX welcome page. Congratulations! You've just deployed your first service packaged as a Helm chart! 

**Step 4:** Modify chart to deploy a custom application 
Now that you've deployed the default NGINX chart, let's make some changes and deploy another application. 

Update `values.yaml` file with the following image keys. 
```
image:
  repository: prydonius/todo
  tag: 1.0.0
  pullPolicy: IfNotPresent
```

As you develop your chart you should use the built-in `linter` to confirm syntax is correct and that you are following best practices. 
```
helm lint mychart
```
If you do not have any errors you should see output: 
```
==> Linting mychart/
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, no failures
```

Now delete the previous release so we can deploy the new `todo` application. 
```
helm del --purge example 
```

After deleting the previous NGINX release, deploy the new application.
```
helm install --name example mychart --set service.type=LoadBalancer
```

Now get the `External-IP` of the service by running 
```
kubectl get svc example-mychart
```
You will see something like: 
```
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP                                                                PORT(S)          AGE
example-mychart   LoadBalancer   10.100.50.215   ad99ac102daa911e99ab706a07e4e3fa-1225924655.ap-south-1.elb.amazonaws.com   8080:30843/TCP   12m
```

You can see the application is running on port 8080, so to see it in a browser load the `EXTERNAL-IP`:8080

**Step 5:** Package application to share   
Helm makes it easy to share your applications with others. Instead of providing them a directory with a bunch of `YAML` files we can package it all up into a simple `tar.gz`. 
```
helm package mychart
```

Now you will see a new file was created: 
```
ls -l mychart* 

mychart-0.1.0.tgz
```

Great! You have successfully deployed applications from charts, used overrides to change default values at deployment time and built your own application to share with others. 

## Congrats!

