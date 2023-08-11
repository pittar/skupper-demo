# Red Hat Service Interconnect (Skupper) Demo

This is a simple use case designed to show the power and simplicity of [Red Hat Service Interconnect](https://www.redhat.com/en/technologies/cloud-computing/service-interconnect) / [Skupper](https://skupper.io) when it comes to connecting services across Kubernetes clusters in a way that doesn't require managing certificates in pods, passthrough routes, TLS SNI, or any other changes to your workloads.

## The Demo

This demo uses the trusty old "Pet Clinic" application.  However, the twist is that the front end application is in one Red Hat OpenShift clsuter, while the MySQL database is in another.  Skupper is used to seamlessly connect the frontend to the database without any changes to the application, no need to manage certificates and passthrough routes, yet still using the principles of "zero trust networking".

If you want to try this demo yourself, but you only have one cluster to work with - no problem!  You can still simply use two namespaces (frontend and database) in the same cluster.  Just ignore the instructions to login to the 2nd cluster.

## Initial Setup

Install the "Red Hat Service Interconnect" operator in both clusters using the OperatorHub interface in the OpenShift Admin UI.  Accept all defaults when installing the operator.

## Cluster 1 / "Site A" - Deploy the front end and the main Skupper Router

Login to the first cluster (Site A) and do the following:

Using the `oc` or `kubectl` cli, deploy the front end app and skupper router:

```
oc apply -k frontend
```

What just happened?

1. A new namespace named "frontend" is created.
2. A `Deployment` is created for the petclinic application.
3. A `ConfigMap` is created with the details needed to setup the Skupper router.
4. A `Secret` is created that the Skupper router will populate with connection details.

A quick look at the `ConfigMap`:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: skupper-site
data:
  name: demo-site-a
  console: "true"
  console-user: "admin"
  console-password: "password"
  flow-collector: "true"
```

The RHSI operator will see this ConfigMap and use the data to configure a Skupper router.  In this case, it will also enable the Skupper console (web UI).

The `Secret` is very minimal.  The important part is the label.  The RHSI operator populate this Secret with certificates and other metadata required to make a connection from another Skupper router in a different cluster (or namespace).

```
apiVersion: v1
kind: Secret
metadata:
  labels:
    skupper.io/type: connection-token-request
  name: skupper-secret
```

Once the Skupper router is running, you can use the [Skupper cli](https://skupper.io/docs/cli/index.html) to check the status and get the console url:

```
oc project frontend
skupper status
```

There is one last thing to do while logged in to this cluster - export the secret!

```
mkdir -p token
oc get secret skupper-secret -o yaml > token/token.yaml
```

Edit this yaml file with your editor of choice and delete the `namespace` line.

This will be used to connect the Skupper routers in the next step.

## Cluster 2 / "Site B" - Deploy the database and a Skupper Router

Login to the second cluster (Site B) and do the following:

Using the `oc` or `kubectl` cli, deploy the front end app and skupper router:

```
oc apply -k database
```

This will:

1. Create a new namespace named "database".
2. A `Deployment/Secret/PVC` is created for the database.
3. A `ConfigMap` is created with the details needed to setup the Skupper router.

Now, apply the secret you exported in the last step to this cluster.

```
oc apply -f token/token.yaml -n database
```

This will tell the Skupper router how to connect to the skupper router in your other cluster.  It has everything it needs (certificates, url, etc...).

Done!  Your routers are now connected.

At this point, you might be wondering how Skupper knows where to route traffic.  This is accomplished with `annotations` on a `Deployment`.  If you check the Deployment for the MySQL database, you will find the following annotations:

```
  annotations:
    skupper.io/address: petclinicdb
    skupper.io/port: '3306'
    skupper.io/proxy: tcp
```

This tells the skupper router that's running in the same namespace that it should expose a service named `petclinicdb` through port `3306` using `tcp`.

Time to switch back over to the other cluster!

## Connecting the Dots

From the OpenShift UI in Cluster 1, navigate to the "frontend" project.

Scale the Petclinic deployment from 0 to 1.  Once it is up and running, you can access the app and add/find pet owners.  The app is connecting to the MySQL database in the other cluster!

If you inspect the environment variables for the frontend app, you'll see the database URL is set to `petclinicdb`.  This means the app will be looking for a local Kubernetes `Service` called "petclinicdb" and trying to access it over port 3306.

Take a look at the services in the namespace and you will see this does indeed exist!  This wasn't originally created when you ran `oc apply -k frontend`, and you can confirm the service is not in this reporsitory.  Skupper created this service automatically when the routers were linked and it discovered the database deployment annotations.  Neat!  The Petclinic app simply uses this service as if the database pod was also running in the same namespace.  Skupper then takes the traffic and sends it over the L7 tunnel it has created to the MySQL database in the other cluster.  This is all handled for you!  Skupper to the rescue!

If you have opened the admin Skupper console (you can get the url with `skupper status` and use the user/pass defined in the ConfigMap), you can also see the two sites in a nice graph and view the traffic in near real time.

##  Conclusion

To keep it short and sweet - if you need to connect services between Kubernetes clusters in simple and secure manner, Red Hat Service Interconnect (Skupper) is the perfect tool for the job.

