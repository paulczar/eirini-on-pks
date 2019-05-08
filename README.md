# Install Eirini on PKS

> Note: jump through the various files and replace $DOMAIN with your own domain.

## Pre-Requisites

### Set nofiles ulimit on worker nodes

```bash
bosh ssh -d .... worker/...
ulimit -n 1048576
```

### Heapster

```bash
kube apply -f heapster

kubectl create clusterrolebinding heapster --clusterrole cluster-admin --serviceaccount=kube-system:heapster

```

### Helm

```bash
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account=tiller
kubectl -n kube-system delete service tiller-deploy
kubectl -n kube-system patch deployment tiller-deploy --patch '
spec:
  template:
    spec:
      containers:
        - name: tiller
          ports: []
          command: ["/tiller"]
          args: ["--listen=localhost:44134"]
'
```

### cert-manager

these instructions work for me on pks on gcp.  your results may vary. You'll need to create a cluster-issuer etc.  maybe there's a better way to get a cert trusted by everything?  i dunno whatever.

```
k create -n kube-system secret generic \
  google-credentials --from-file=google-credentials.json

helm install --namespace kube-system --name cert-manager \
  --values kube-system/cert-manager/values.yaml \
  stable/cert-manager
kubectl apply -f kube-system/cluster-issuer.yaml
```

## UAA

Install UAA

```bash
helm install --namespace uaa --name uaa --values values.yaml eirini/uaa
```

Update DNS with the IP from the service:

```bash
$ kubectl get svc uaa-uaa-public
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
uaa-uaa-public   LoadBalancer   10.100.200.209   35.239.4.74   2793:31389/TCP   4m28s
```

Load UAA ca-cert from ^

```bash
SECRET=$(kubectl get pods --namespace uaa -o jsonpath='{.items[?(.metadata.name=="uaa-0")].spec.containers[?(.name=="uaa")].env[?(.name=="INTERNAL_CA_CERT")].valueFrom.secretKeyRef.name}')
CA_CERT="$(kubectl get secret $SECRET --namespace uaa -o jsonpath="{.data['internal-ca-cert']}" | base64 --decode -)"
```

## Install scf/eirini

Create a `scf` namespace:

```
kubectl create namespace scf
```

Create certs for BITS:

> Note this needs to be a root trusted ca ... easiest way to do it is via lets-encrypt ... you'll need to modify the cert manifest below to suit your env:

```
kubectl apply -f bits-certs.yaml
```

helm install --namespace scf --name scf \
  --values values.yaml \
  --set "secrets.UAA_CA_CERT=${CA_CERT}" \
  eirini-release/helm/cf

update DNS with loadbalancer services

```
$ kubectl get svc | grep Load
bits                                        LoadBalancer   10.100.200.43    35.192.160.194   6666:30411/TCP                                                                                                                                    71s
diego-ssh-ssh-proxy-public                  LoadBalancer   10.100.200.44    <pending>        2222:31132/TCP                                                                                                                                    70s
router-gorouter-public                      LoadBalancer   10.100.200.32    <pending>        80:30275/TCP,443:30687/TCP,4443:32052/TCP                                                                                                         70s
tcp-router-tcp-router-public                LoadBalancer   10.100.200.112   <pending>        20000:30355/TCP,20001:30102/TCP,20002:31527/TCP,20003:30779/TCP,20004:31208/TCP,20005:31251/TCP,20006:30106/TCP,20007:32191/TCP,20008:30630/TCP   70s
```

## test CF

```
cf api --skip-ssl-validation https://api.app.$DOMAIN

cf login
Email> admin
Password> ********

cf create-space demo

cf target -s demo

git clone https://github.com/cloudfoundry-samples/cf-sample-app-spring

cd cf-sample-app-spring

cf push demo

```

## Install Stratos

> note: I had to create a special storage class as gcp kept scheduling two PVCs for one pod in different zones.

```bash
helm repo add stratos-ui https://cloudfoundry-incubator.github.io/stratos

helm install stratos-ui/console \
    --namespace stratos --name stratos \
    --values stratos/values.yaml --set "storageClass=us-central1-c"
```

update DNS with IP:

hostname will be `stratos.app...`

```
kubectl get svc
NAME              TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)         AGE
stratos-mariadb   ClusterIP      10.100.200.119   <none>           3306/TCP        2m41s
stratos-ui-ext    LoadBalancer   10.100.200.157   104.197.79.192   443:30312/TCP   2m41s
```