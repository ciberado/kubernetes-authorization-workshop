# Kubernetes authentication deep dive

## Preparation

* Define a namespace for the workshop

```bash
kubectl create ns demo
kubectl config set-context --namespace demo --current
```

## Basic concepts

* Look at the `default` *namespace* to find the *service* connected to the *api-server*

```bash
kubectl get services -n default
```

* Use the *very verbose* mode of `kubectl` for understanding wich path is hitting each type of request, what are the *api-groups* and the *resource* names

```bash
kubectl run mypod --image=nginx --restart=Never -v 7
kubectl create service clusterip mysvc --tcp 80 -v 7
kubectl create deployment --image nginx mydeployment -v 7
kubectl create job --image bash  myjob -v 7
```

* Instead of using the familiar friendly syntax, you can actuall send request directly to those API paths

```bash
kubectl get --raw /api/v1/pods
kubectl get --raw /api/v1/pods | jq '.items[].metadata | "\(.name) (\(.namespace))"' -r
kubectl get --raw /apis/apps/v1/namespaces/kube-system/deployments | jq
kubectl get --raw /apis/apps/v1/namespaces/kube-system/deployments | \
  jq '.items[].metadata.name'
```

## Investigate the *pod* identity

* Look at the details of the *pod*: there a reference to a `default` *service account*, even if we didn't asked for it

```bash
kubectl get pod mypod -ojson | jq -C | less -r
```

* Where is that service account? Turns out, a default *service account* is created for each *namespace*, and it describes the security identity of the *pod*

```bash
kubectl get serviceaccounts
```

* Getting the details of the *service account* we realize there is a *secret* involved, also created automatically when the *service account* is added to the *namespace*

```bash
kubectl describe sa default
```

* Let's invest some time checking the content of the secret

```bash
kubectl get secrets
```

* We will need more details, so

```bash
SECRET_NAME=$(kubectl get secrets -ojsonpath="{.items[0].metadata.name}"); echo $SECRET_NAME
kubectl get secret $SECRET_NAME -ojson | jq -C | more
```

* Now we can dive in the `data` content. The `namespace` value does not look very informative. Maybe is it encrypted?

```bash
kubectl get secret $SECRET_NAME -ojsonpath="{.data.namespace}"; echo
```

* It is not! Just codified using `base64`

```bash
kubectl get secret $SECRET_NAME -ojsonpath="{.data.namespace}" | base64 --decode ; echo
```

* The `namespace` attribute wasn't very exciting, but the rest of them looks much more interesting! Let's see the *Certification Authority* certificate

```
kubectl get secret $SECRET_NAME -ojsonpath="{.data.ca\.crt}" | base64 --decode ; echo
```

* Of course, it can be examined in detail (try to find the public key used to validate the signatures that will appear later)

```bash
kubectl get secret $SECRET_NAME -ojsonpath="{.data.ca\.crt}" | \
  base64 --decode | \
  openssl x509 -text
```

* Our last field is actually a [Json Web Token](https://jwt.io/introduction). Take note of the `service-account-uid` value, as it will appear later

```bash
JWT=$(kubectl get secret $SECRET_NAME -ojsonpath="{.data.token}" | base64 --decode)
jq -R 'split(".") | .[0],.[1] | @base64d | fromjson' <<< $(echo "${JWT}")
```

## Identity injection

* Let's look again at our *pod*, focusing in the *volume* mounted in the *container*

```bash
kubectl get pod mypod -ojson | jq -C | less -r
```

* A *projected volume* generates files from different sources under a single directory structure. The `serviceAccountToken` is injected by the [Service Account Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#bound-service-account-token-volume), and the `downwardsAPI` is a standard way of injecting metadata in the *pod*

* The `configMap` refers to a resource in the `kube-system` namespace

```bash
kubectl get configmap kube-root-ca.crt -n kube-system
```

```bash
kubectl describe configmap kube-root-ca.crt -n kube-system
```
* Take a look at the location of the mounted *volume*

```bash
kubectl get pod mypod -ojsonpath="{.spec.containers[].volumeMounts[].mountPath}";echo
```

## Inside the *pod*

* Jump into the pod we created before. From now, we are executing commands **inside of `mypod`**

```bash
kubectl exec -it mypod  -- bash
```

* Look at the content of the *projected volume*

```bash
cd /var/run/secrets/kubernetes.io/serviceaccount
ls
```

* The content of the *CA* is what we expected

```bash
cat ca.crt
```

* And we can look at the details of the *JWT*

```bash
cat token
```

* We will need some extra tooling

```bash
apt update; apt install jq less -y
```

* Now we can look at it:

```bash
jq -C -R 'split(".") | .[0],.[1] | @base64d | fromjson' <<< $(cat token) | less -r
```

* Let's try to call the *API server* without any kind of authentication, using the `kubernetes` service (we will fail because we are considered `anonymous`)

```bash
curl -s \
  --insecure \
  -X GET \
  https://kubernetes.default.svc/api/v1/namespaces/demo/pods \
  | jq
```

* Ok, we will try again. But this time we will build our authorization token, just like any Kubernetes library does:

```bash
SERVICE_ACCOUNT_VOL=/var/run/secrets/kubernetes.io/serviceaccount
TOKEN=$(cat ${SERVICE_ACCOUNT_VOL}/token)
CACERT=${SERVICE_ACCOUNT_VOL}/ca.crt

NAMESPACE=$(cat ${SERVICE_ACCOUNT_VOL}/namespace)
API_SERVER=https://kubernetes.default.svc
API_PATH=/api/v1/namespaces/$NAMESPACE/pods

curl -s \
  --cacert ${CACERT} \
  --header "Authorization: Bearer ${TOKEN}" \
  -X GET \
  ${API_SERVER}${API_PATH}
```

* Yes, we have failed again. But this time the systems acknowledges our identity... but we do not have enought permissions. So it will better to leave the *pod*

```bash
exit
```

* This *pod* is not useful for us, as it is not associated to a powerful *service account*. Delete it with

```bash
kubectl delete pod mypod
```

## Creating custom *Service Accounts*

* Define and create the new *Service Account* for our workload. It is very common to create a *service account* for each type of *pod* that requires specific permissions

```bash
cat << EOF > my-pod-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-pod-sa
EOF	

kubectl apply -f my-pod-sa.yaml
```

* Now, generate the list of permissions (including references to the API `groups`, the kind of `resources` and the `verbs` that will be allowed. We are going to limit the permissions to resources in a particular *namespace*, so we will use a *Role*. If the permissions should be granted for cluster-wide resources (like *persitent volumes*, *nodes*, etc) we would have needed a *ClusterRole* instead

```bash	
cat << EOF > read-pods-and-depl-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1 
metadata:
    name: read-pods-and-depl-role
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments"]
  verbs: ["get", "list", "watch"]
EOF

kubectl apply -f read-pods-and-depl-role.yaml
```

* Now, let's link the *role* with the *service account* using a *rolebinding*, restricted to our current namespace. To provide cluster-wide permissions, we should use a *ClusterRoleBinding* instead

```bash
cat << EOF > my-pod-sa-list-pods-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-pod-sa-list-pods-rolebinding
subjects:
- kind: ServiceAccount
  name: my-pod-sa
roleRef:
  kind: Role
  name: read-pods-and-depl-role
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f my-pod-sa-list-pods-rolebinding.yaml
```

* There is an old spell to list the permissions assigned to a particular *service account*

```bash
kubectl get rolebinding,clusterrolebinding \
  --all-namespaces \
  -ojsonpath='{range .items[?(@.subjects[0].name=="my-pod-sa")]}[{.roleRef.kind},{.roleRef.name}]{end}'; \
  echo
```

* We are almost done: execute the *pod* again, this time with its new security identity

```bash
cat << EOF > my-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  serviceAccountName: my-pod-sa
  containers:
  - name: mypod
    image: nginx
EOF

kubectl apply -f my-pod.yaml
```

## Authorization check from inside the *pod*

* We will need to jump again into the *pod*. From here, the rest of the commands of the section will be run **inside** the main *container* of the *pod*

```bash
kubectl exec -it mypod  -- bash
```

* Now we will be able to retreive the list of *pods* in the namespace

```bash
apt update; apt install jq less -y
```

```bash
SERVICE_ACCOUNT_VOL=/var/run/secrets/kubernetes.io/serviceaccount
TOKEN=$(cat ${SERVICE_ACCOUNT_VOL}/token)
CACERT=${SERVICE_ACCOUNT_VOL}/ca.crt

NAMESPACE=$(cat ${SERVICE_ACCOUNT_VOL}/namespace)
API_SERVER=https://kubernetes.default.svc
API_PATH=/api/v1/namespaces/$NAMESPACE/pods
```

```
curl -s \
  --cacert ${CACERT} \
  --header "Authorization: Bearer ${TOKEN}" \
  -X GET \
  ${API_SERVER}${API_PATH} \
  | jq -C \
  | less -r
```
