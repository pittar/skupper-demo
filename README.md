# Skupper Demo

Deploy frontend in "DC1":

```
oc login <cluster 1>
oc apply -k frontend -n frontend
```

Initialize Skupper in "DC1" and create a token.

```
skupper init --site-name dc1 -n frontend
skupper token create dc1.yaml -n frontend
```

Deploy database in "DC2":

```
oc login <cluster 2>
oc apply -k database -n database
```

Initialize Skupper in "DC2" and link "DC1":

```
skupper init --site-name dc2 -n database
skupper link create dc1.yaml -n database
```

Create the scupper service and bind it to the MySQL deployment.

```
skupper service create petclinicdb 3306 -n database
skupper service status -n database
skupper service bind petclinicdb deployment petclinicdb -n database
```

You should be ready to scale up the frontend!


