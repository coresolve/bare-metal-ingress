## Problem Description:

The default tectonic ingress controller will start an pod on each worker node and bind to port 443 and port 80.
We would like to create a different ingress controller and be able to bind to those ports.

## Understand the current state:
We need to update the pattern used by the tectonic ingress controller. 

Currently, tectonic-ingress-controller will place controller on any node that doesn't have the label "master"

You can see this in your cluster by running:

`kubectl get ds --namespace=tectonic-system tectonic-ingress-controller -o yaml`

Part of that will describe the affinity settings:
```yaml
      annotations:
        scheduler.alpha.kubernetes.io/affinity: |
          {
            "nodeAffinity": {
              "requiredDuringSchedulingIgnoredDuringExecution": {
                "nodeSelectorTerms": [
                  {
                    "matchExpressions": [
                      {
                        "key": "master",
                        "operator": "DoesNotExist"
                      }
                    ]
                  }
                ]
              }
            }
          }
```
This match is equivelent to:

`kubectl get nodes -l '!master'`

This will give you a list of nodes that are "workers"
```livescript
$ kubectl get node -l '!master'
NAME                            STATUS    AGE
worker001.bmetal.mauilion.com   Ready     10d
worker002.bmetal.mauilion.com   Ready     10d
worker003.bmetal.mauilion.com   Ready     10d
```

## Create new labels:
We need to change the logic here. So we will label the first worker `tectonic-ingress=true`. To do this I am using the jsonpath expression to return the first node where it's labels don't have `master` defined.

`kubectl label node $(kubectl get node -l '!master' -o jsonpath={.items[0].metadata.name}) tectonic-ingress=true`

This should tell us that one of the worker nodes has been labeled:
```livescript
$ kubectl label node $(kubectl get node -l '!master' -o jsonpath={.items[0].metadata.name}) tectonic-ingress=true
node "worker001.bmetal.mauilion.com" labeled
```

Now we will label the rest of the nodes with `my-ingress=true`
```livescript
$ kubectl label node -l '!master,!tectonic-ingress' my-ingress=true
node "worker002.bmetal.mauilion.com" labeled
node "worker003.bmetal.mauilion.com" labeled
```


Now that are labels are in place we can start doing the fun stuff.

## Change the tectonic-ingress-controller daemonset:
Let's see the controller pods that are running: 

```livescript
$ kubectl get po --namespace=tectonic-system -l app=tectonic-lb
NAME                                READY     STATUS    RESTARTS   AGE
tectonic-ingress-controller-7741f   1/1       Running   0          10d
tectonic-ingress-controller-gpl0c   1/1       Running   0          10d
tectonic-ingress-controller-l8hlh   1/1       Running   0          10d
```

Then we will patch the tectonic-ingress-controller daemonset with:

```livescript
$ kubectl apply -f update-tectonic/patched-tectonic-ingress-ds.yaml
daemonset "tectonic-ingress-controller" configured
```
Now we should see the ds reduce to one pod.

```livescript
$ kubectl get po --namespace=tectonic-system -l app=tectonic-lb
NAME                                READY     STATUS    RESTARTS   AGE
tectonic-ingress-controller-7741f   1/1       Running   0          10d
```

If you take a look at the patched-tectonic-ingress-ds.yaml file you can see that the selector logic has changed to select only those nodes with tectonic-ingress defined.

## Create our own ingress-controller:
Now we can begin setting up our own ingress.

cd into the my-ingress folder and create the resources:

```livescript
$ cd my-ingress
$ kubectl create -f .
replicationcontroller "default-http-backend" created
service "default-http-backend" created
daemonset "nginx-ingress-controller" created
configmap "nginx-load-balancer-conf" created
```

This creates a basic nginx ingress listening on port 80 on any node that has the label my-ingress defined.
You can inspect the `nginx-ingress-controller_ds.yaml` file for the match selector to understand that part.

The other files will create a default-http-backend with will serve as a fallback for anything that an ingress doesn't match and the configmap sets up the config for the nginx contoller pods.

to check on the status of these resources you can:

```livescript
$ kubectl get rc,svc,ds -l my-ingress -n kube-system
NAME                      DESIRED   CURRENT   READY     AGE
rc/default-http-backend   1         1         1         10m

NAME                       CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
svc/default-http-backend   10.3.0.92    <nodes>       80:31651/TCP   10m

NAME                          DESIRED   CURRENT   READY     NODE-SELECTOR   AGE
ds/nginx-ingress-controller   2         2         2         <none>          10m
```

## Create a test-app and test it.
Now we can test this ingress with a test-app.

```livescript
$ cd ../test-app
$ kubectl create -f .
deployment "echoserver" created
ingress "echoserver-ingress" created
service "echoserver" created
```
To check on the status of these resources you can:

```livescript
$ kubectl get all -l run=echoserver
NAME                            READY     STATUS    RESTARTS   AGE
po/echoserver-308202803-xmncz   1/1       Running   0          8s

NAME             CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
svc/echoserver   10.3.0.208   <nodes>       8080:30476/TCP   8s

NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/echoserver   1         1         1            1           8s

NAME                      DESIRED   CURRENT   READY     AGE
rs/echoserver-308202803   1         1         1         8s
```

Now to determine the ip addresses that our new ingresses are exposed on.

```livescript
$ kubectl describe ingress echoserver-ingress
Name:			echoserver-ingress
Namespace:		default
Address:		10.0.0.51,10.0.0.52,10.0.0.52
Default backend:	default-http-backend:80 (<none>)
Rules:
  Host			Path	Backends
  ----			----	--------
  echo.me
    			/ 	echoserver:8080 (10.2.0.34:8080)
Annotations:
  rewrite-target:	/
Events:
  FirstSeen	LastSeen	Count	From				SubObjectPath	Type		Reason	Message
  ---------	--------	-----	----				-------------	--------	------	-------
  1m		1m		1	{nginx-ingress-controller }			Normal		CREATE	default/echoserver-ingress
  1m		1m		1	{nginx-ingress-controller }			Normal		CREATE	ip: 10.0.0.51
  1m		1m		3	{nginx-ingress-controller }			Normal		UPDATE	default/echoserver-ingress
  1m		1m		1	{nginx-ingress-controller }			Normal		CREATE	default/echoserver-ingress
  1m		1m		3	{nginx-ingress-controller }			Normal		UPDATE	default/echoserver-ingress
  1m		1m		1	{nginx-ingress-controller }			Warning		UPDATE	error: Operation cannot be fulfilled on ingresses.extensions "echoserver-ingress": the object has been modified; please apply your changes to the latest version and try again
  1m		1m		2	{nginx-ingress-controller }			Normal		CREATE	ip: 10.0.0.52
```

In the above you can see the line `Address:		10.0.0.51,10.0.0.52,10.0.0.52` This means that our ingress controller is listening on port 80 on those ip addresses.

To test this you can add the following to your hosts file:

```bash
10.0.0.52 echo.me this.me
```

**You will want to use your _own address_ here not 10.0.0.52**

However, in my case a curl shows the output from our echoserver app.

```livescript
$ curl echo.me
CLIENT VALUES:
client_address=10.2.3.0
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://echo.me:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
connection=close
host=echo.me
referer=
user-agent=Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Trident/5.0)
x-forwarded-for=10.0.0.254
x-forwarded-host=echo.me
x-forwarded-port=80
x-forwarded-proto=http
x-real-ip=10.0.0.254
BODY:
-no body in request-
```

and a curl to this.me (undefined and will fallback to the `default-http-backend` service. I get redirected to the tectonic console.
```livescript
$ curl -Lk this.me

--- SNIP ---

      window.SERVER_FLAGS = {"k8sAPIVersion":"v1","authDisabled":false,"kubectlClientID":"tectonic-kubectl","basePath":"/","loginURL":"https://tectonic.bmetal.mauilion.com/auth/login","loginSuccessURL":"https://tectonic.bmetal.mauilion.com/","loginErrorURL":"https://tectonic.bmetal.mauilion.com/error","logoutURL":"https://tectonic.bmetal.mauilion.com/auth/logout"};

--- SNIP ---
```

## Profit!

At this point it's good to understand that if you take the worker that is hosting the tectonic-ingress-controller down. You will be unable to connect to the ui. In a cluster with enough nodes you might ensure that more than one node is given the `'tectonic-ingress=true'` label.

You will still need to configure something reasonable in DNS. For our test-app we are using /etc/hosts. In reality you would want to create something like \*.yourapp.com and point that to the addresses that your ingress controller uses.

To see the addresses for our ingress you can: 
```livescript
$ kubectl get ingress echoserver-ingress -o jsonpath='{range .status.loadBalancer.ingress[*]}{.ip}{","}{end}{"\n"}'
10.0.0.52,10.0.0.51,10.0.0.51,
```
