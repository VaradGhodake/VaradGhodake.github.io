### Please fix this: k8s edition

<br />

☸️🛠️

---

Definitely not exhaustive. Just some of the things one should start off with debugging a problem within their k8s cluster. <br />
Alright, split your window in two with `tmux` to start things off. We'll use one to test things within a pod and other for general purpose debugging.

Also, lot of these commands need namespace specification, so make sure you add `-n {namespace}` or create an alias

#### Get a busybox container inside the namespace

Shares the same network namespace, creates a new pod that gets destroyed once exit.

```sh
# busybox inside the pod
kubectl run -it --rm --restart=Never busybox --image=gcr.io/google-containers/busybox sh
```

###### For pods:

```sh
# sort events by timestamp
kubectl get events --sort-by='.metadata.creationTimestamp'

# current container logs
kubectl logs ${POD_NAME} ${CONTAINER_NAME}

# previously crashed container's logs
kubectl logs --previous ${POD_NAME} ${CONTAINER_NAME}

# get a shell for monocontainer
kubectl exec -it -p ${POD_NAME} -c {CONTAINER_NAME} -- sh

# create a debug container that shares the process namespace with a **different image**
kc get pods {POD_NAME} -o jsonpath='{.spec.containers[*].name}'
kc debug -it {POD_NAME} --image={NEW_IMAGE} --target={CONTAINER_NAME}

# create a debug container with the **start command changed**
# Get container names with JsonPath
kc get pods {POD_NAME} -o jsonpath='{.spec.containers[*].name}'
kubectl debug {POD_NAME} -it --copy-to={CONTAINER_NAME}-debug --container={CONTAINER_NAME} -- sh
```


###### Services

Labels for the pod and selectors for service spec should match, obviously

```sh

# expose ports from a deployment
kubectl expose deployment hostnames --port=<SERVICE_PORT> --target-port={CONTAINER_PORT}

# port check from busybox
wget -qO- $IP:$PORT

# hostname check from busybox
wget -O- $HOSTNAME

# /etc/resolv.conf should contain cluster nameserver address
# nslookup hostnames
# Server:    <DNS IP>
# Address 1: <DNS IP> <DNS_HOST_NAME>
# 
# Name:      hostname-ip
# Address 1: <hostname-ip> hostname.<namespace>.svc.cluster.local
```

From outside:

```sh
# check if the DNS works
nslookup kubernetes.default

# check if the endpoints are set
kubectl get endpoints hostnames

# kube-proxy debugging: it's a service on the node
ps auxw | grep kube-proxy
```


###### Ingress

Good resource for nginx ingress: https://kubernetes.github.io/ingress-nginx/troubleshooting/

Generally, devs like to schedule their ingress pods in another namespace. Find it, grep! <br />
It is a *pod* afterall. Check events and describe it to get more information.

```sh
kubectl get ingress -o wide | grep "blah blah"

# describe and check if the Host, Path, Backends are right
kubectl describe ingress {INGRESS_CONTROLLER}

# read the generated nginx config
kubectl exec -it {INGRESS_CONTROLLER} -- cat /etc/nginx/nginx.conf
```

