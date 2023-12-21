# Workshop Service Mesh.

## Install Terminal

Use the admin login/password and OpenShift url given by your instructor. 

I suggest to install also the web terminal operator to run some commands from the OpenShift Cloud Shell:

* Web Terminal

## Create a project for the BookInfo application

In the terminal, type:

```bash
oc new-project bookinfo

oc apply -f https://raw.githubusercontent.com/danielpenagos/samples-servicemesh/main/bookinfo.yaml
```

Review the official documentation of the application at:
[istio doc](https://istio.io/docs/examples/bookinfo/)

Once the application is running, review all the resources:

```bash
oc get all
```

Expose the productpage deployment:

```bash
oc expose service productpage
```

## Install Service Mesh Operators

Use the admin login/password and OpenShift url given by your instructor. 

Use the Administrator perspective and install the following operators from the OperatorHub.

* OpenShift Elasticsearch Operator
* Red Hat OpenShift distributed tracing platform
* Kiali Operator (from Red Hat)
* Red Hat OpenShift Service Mesh

## Deploy Service Mesh Control Plane

It is a good practice to install the service control plane in a separate project, other than the application one. 
So, we need to create another project:

```bash
oc new-project bookretail-istio-system
```

To install the control plane: 
- Go to the service mesh operator already installed.
- Go to the Service Mesh Control Plane tab.
- Select the project bookretail-istio-system.
- Click on the create button. Select all the default parameters.
- Go to the topology view of the bookretail-istio-system and wait until all the components are ready. **The last one to be ready is kiali.**

## Enroll the application project to the Control Plane. 

To enroll your app project to be shown in the tools:
- Go to the service mesh operator already installed.
- Go to the Service Mesh Member Roll tab.
- Click on the create button.
- Include the bookinfo project in the list in the yaml.  

```yaml
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
  namespace: bookretail-istio-system
spec:
  members:
    # a list of projects joined into the service mesh
    - bookinfo
```
At this point you can open kiali and review the graph to review all the services. In the service list, review that all of the services have missing sidecars. Review the telemetry of each service.

## Inject sidecars to the application. 
Each Pod should have its own sidecar to send information to the servicemesh control plane. 
For each deployment we need to include the following annotation:

```yaml
spec:
...
 template:
   metadata:
     annotations:
       sidecar.istio.io/inject: 'true'
```

After updating each yaml definition, check the service list in kiali to review that the missing sidecar alert is not visible anymore. 
Generate some traffic, navigating the application front-end.
Review the telemetry of each service.
Review also the kiali console to see the mesh diagram.

**Note**: To have new pods in the namespace to inherit this feature, you only need to include this label in the project/namespace.
```bash
oc label namespace bookinfo istio-injection=enabled
```

At this point you will be able to review the graph with the mesh topology if you generate some traffic. 
Notice the source is labeled as "unknown"


## Create gateway and virtual service.


Create these elements with the following command:
```bash

cat << EOF | oc create -n bookinfo -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
EOF
```

Review the url just created
```bash

export GATEWAY_URL=$(oc get -n bookretail-istio-system route istio-ingressgateway -o jsonpath='{.spec.host}')

export ROUTER_URL=$(oc get -n bookinfo route productpage -o jsonpath='{.spec.host}')

echo $GATEWAY_URL

echo $ROUTER_URL

```

## Create destination rules.
Let's configure another virtual service, this time for a microservice that is going to have a new version:

```bash
oc project bookinfo
cat << EOF | oc create -n bookinfo -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 80
    - destination:
        host: reviews
        subset: v2
      weight: 20
EOF
```

In this case, we need to configure the set of destination rules, to identify the versions of each pod.

```bash
cat << EOF | oc create -n bookinfo -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v2-mysql
    labels:
      version: v2-mysql
  - name: v2-mysql-vm
    labels:
      version: v2-mysql-vm
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
---
EOF
```


Generate some traffic to test the configuration.

```bash
export GATEWAY_URL=$(oc -n bookretail-istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')

echo "http://$GATEWAY_URL/productpage"

while true; do curl -sSL "http://$GATEWAY_URL/productpage" | head -n 5; sleep 1; done 
```