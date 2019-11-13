# Running PostgreSQL on K8s

This repo contains scripts and manifests to test deployments of a [PostgreSQL](https://www.postgresql.org) cluster on
[Kubernetes](https://k8s.io).

## Lessons learned

* Minikube is a really nice tool to experiment with Kubernetes. It works great (out-of-the-box) on Linux systems with
  VirtualBox as hypervisor.
* Learning Kubernetes is not easy. It takes a lot of time
* Minikube does not return the service URL and port when the K8s service is located in a non-default namespace. You need 
  to add the global flag `-n {namespace}` after `minikube service` to start a Minikube load balancer.
* Documentation of both solutions is rather concise. The basic setup of the Crunchy Data stack does not include the PostGIS
  extension. I did not manage to deploy PostGIS by just changing the Helm chart values (did we hit a bug?).
* In case of problems with deploying a new cluster: always look in the log file of the postgres operator. It usually tells
  why a PostgreSQL Pod could not be started.
* Load balancing postgres connections using pgPool-II works quite well.
* Self-healing after failure of a postgres master node on Kubernetes works within 10 seconds (if the transaction log of the
  replica was processed before failure).

## Preparations

All code should run on Windows/Linux/Mac. It is tested on Ubuntu 18.04 and 19.04.

### Install Minikube

Minikube is a single node installation of Kubernetes which is useful for testing. It includes a GUI (dashboard) to get
a better overview of the state of the cluster. See https://kubernetes.io/docs/setup/learning-environment/minikube/ for
details on the installation of Minikube. The installation takes a while, because VM images and Docker images must be
downloaded.

After installation run the follow commands (starting Minikube takes at least 3-5 minutes):

    minikube start
    minikube dashboard

The dashboard opens on a random port and opens a page/tab in your default browser.

### Executing command with Kubectl

When running multiple K8s clusters (e.g. Minikube and on Google Kubernetes Engine/AHP v2) you must switch between configurations
in order to use the correct connection with the K8s API server. Follow the next steps to use Kubectl on your machine with
multiple clusters. For Linux users: use the tips on the [cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
to make working with Kubectl even easier!

Create files under `~/.kube/` that hold connection configurations for GKE and Minikube:

* `~/.kube/config`
* `~/.kube/config-ahp`

You can switch from local (minikube) to GCK and back with:

    kubectl config use-context CONTEXT_NAME

to list all contexts:

    kubectl config get-contexts

To define the default namespace:

    kubectl config set-context $(kubectl config current-context) --namespace=NAMESPACE
    
### Install Helm

In order to deploy the necessary K8s operators Helm is used. Follow these steps to install helm on the cluster:

    helm init --client-only
    helm plugin install https://github.com/rimusz/helm-tiller
    
## Zalando stack

The following commands follow the instructions from the [Zalando Postgres Operator Quickstart manual]
(https://postgres-operator.readthedocs.io/en/latest/quickstart/). All manifest files are in the folder `zalando` of this
repository. They are based on the files from the *Zalando Postgres Operator* repository (see step 1 belog). The cluster
is named `geodanmaps-orggeodb-cluster` and has 2 nodes. Note that we don't refer to any specific namespace in the following
commands, so all changes will be applied in the default namespace (dependening on your Kubectl configuration; see above).
If you plan to deploy multiple clusters, you must deploy each cluster in its own namespace or change the name of the
deployment in the files `zalando/orggeodb_cluster.yaml` and `zalando/deploy_pgpool2.yaml`.

1. `git clone https://github.com/zalando/postgres-operator.git`
2. `cd postgres-operator`
3. Deploy the operator using helm:

    helm tiller run helm install --name zalando charts/postgres-operator
    
4. Deploy a minimal (2-node) PostgreSQL cluster (use the file from this repo): 

    kubectl create -f zalando/orggeodb_cluster.yaml

5. Deploy a simple setup of pgPool-II for load balancing and connection pooling:

    kubectl create -f zalando/deploy_pgpool2.yaml

## Working with psql

In order to connect to the connection pool, you must create a proxy to the K8s API server. Since `psql` uses TCP traffic
we need *port forwarding* to get the job done:

    kubectl port-forward service/geodanmaps-orggeodb-cluster-pgpool2 5433:5432
    
This command opens port `5433` on the local loopback device (i.e. `localhost`) and forwards all trafic to one of the
running pods with pgPool-II.
    
Use the following bash commands to set [environment variables](https://www.postgresql.org/docs/11/libpq-envars.html) to 
make connecting with PostgreSQL easier:

    export PGUSER=orggeodbadmin
    export PGHOST=localhost
    export PGPORT=5433
    export PGPASSWORD=$(kubectl get secret orgeodbadmin.geodanmaps-orggeodb-cluster.credentials -o 'jsonpath={.data.password}' | base64 -d)
    
You can now connect to the database from your local machine using a simple `psql` command:

    psql orggeodb

### Connecting to the database without pgPool-II

We can connect to either the master or the replica via the Minikube load balancer (in production life this would be a proper
load balancer). To get the IP address an port of the load balancer, use the command `minkube service -n zalando list`:

    |-----------|-----------------|-----------------------------|
    | NAMESPACE |      NAME       |             URL             |
    |-----------|-----------------|-----------------------------|
    | zalando   | gm-cluster      | http://192.168.99.102:32436 |
    | zalando   | gm-cluster-repl | http://192.168.99.102:30593 |
    |-----------|-----------------|-----------------------------|

Since the password of the postgres user was already retrieved from the corresponding K8s secret (see step 6 above) we can
simply connect to the replicas using the following command: `psql -U postgres -h 192.168.99.102 -p 30593`

### Simulating fail-over

We simulate fail-over by removing the pod which hosts the master using the command ``. Before issuing this command we setup
a watcher that shows the removal of a pod, master election (node 2) and creation of a new pod:

    NAME           READY STATUS              RESTARTS   AGE   SPILO-ROLE   LABELS
    gm-cluster-0   1/1   Terminating         0          84m   master       application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,spilo-role=master,statefulset.kubernetes.io/pod-name=gm-cluster-0,team=GM
    gm-cluster-0   1/1   Terminating         0          84m                application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,statefulset.kubernetes.io/pod-name=gm-cluster-0,team=GM
    gm-cluster-1   1/1   Running             0          84m   replica      application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,spilo-role=replica,statefulset.kubernetes.io/pod-name=gm-cluster-1,team=GM
    gm-cluster-2   1/1   Running             0          84m   master       application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,spilo-role=master,statefulset.kubernetes.io/pod-name=gm-cluster-2,team=GM
    gm-cluster-2   1/1   Running             0          84m   master       application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,spilo-role=master,statefulset.kubernetes.io/pod-name=gm-cluster-2,team=GM
    gm-cluster-0   0/1   Terminating         0          84m                application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,statefulset.kubernetes.io/pod-name=gm-cluster-0,team=GM
    gm-cluster-0   0/1   Terminating         0          84m                application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,statefulset.kubernetes.io/pod-name=gm-cluster-0,team=GM
    gm-cluster-0   0/1   Terminating         0          84m                application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,statefulset.kubernetes.io/pod-name=gm-cluster-0,team=GM
    gm-cluster-0   0/1   Pending             0          0s                 application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,statefulset.kubernetes.io/pod-name=gm-cluster-0,team=GM
    gm-cluster-0   0/1   Pending             0          0s                 application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,statefulset.kubernetes.io/pod-name=gm-cluster-0,team=GM
    gm-cluster-0   0/1   ContainerCreating   0          0s                 application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,statefulset.kubernetes.io/pod-name=gm-cluster-0,team=GM
    gm-cluster-0   1/1   Running             0          2s                 application=spilo,cluster-name=gm-cluster,controller-revision-hash=gm-cluster-f4c79f6c,statefulset.kubernetes.io/pod-name=gm-cluster-0,team=GM

The whole process takes about 10s, but the election of the new master is done within 2s. *Note: the SPILO-ROLE shows the role of the postgresql node*

### Scaling the database cluster

Adding or removing Postgres nodes is very simple: update the *manifest* file and run the following command:

    kubectl -n zalando replace -f zalando/custom_cluster.yaml
    
### Upgrade postgres major version

Let's say we want to migrate from 9.6 to 10. There's no clear documentation on how to update a running Postgres cluster on
Kubernetes. The Patroni repository has an issue with a [short explanantion](https://github.com/zalando/patroni/issues/424), 
but the targeted setup does not include the postres-operator. However, this would be a good starting point for tests.

## Crunchy Data stack

The crunchy stack can be deployed using a helm chart. The following instructions are taken from the 
[installation manual of Crunchy Data](https://crunchydata.github.io/postgres-operator/stable/installation/#helm-chart).

1. `git clone https://github.com/CrunchyData/postgres-operator.git`
2. `cd chart`
9. Configure the location of the cloned Git repository: `export CC_REPO=/tmp/postgres-operator`
4. Copy missing files from the *develop* branch in order to fix a bug in the Helm chart, see https://github.com/CrunchyData/postgres-operator/issues/618 
   for details. 
3. Create a new namespace: `kubectl create namespace crunchydata`
4. Make sure that Tiller is running: 
    1. `helm init`
    2. `helm init --service-account tiller`
5. Generate keys to connect with the postgres-opearator after deployment: `bash gen-pgo-keys.sh`.
6. Deploy the operator using helm: `helm install -f crunchy_containers/custom_values.yaml --name=postgres-operator --namespace=crunchydata $CC_REPO/chart/postgres-operator`
8. Configure the connection string for the *pgo* client using the IP address of the operator: `export CO_APISERVER_URL=$(minikube service -n crunchydata postgres-operator-pgo --url | sed 's/http/https/g')`
9. Configure the location of the certificates that were generated in step 4:

       export PGO_CA_CERT=$CC_REPO/chart/postgres-operator/files/apiserver/server.crt
       export PGO_CLIENT_CERT=$CC_REPO/chart/postgres-operator/files/apiserver/server.crt
       export PGO_CLIENT_KEY=$CC_REPO/chart/postgres-operator/files/apiserver/server.key   

11. Download version 3.5.0 of the pgo binary from https://github.com/CrunchyData/postgres-operator/releases
12. Test that the client is working: `pgo version`. The client and server versions must match. If not, download the correct
    version of the client (see step 11).
    
### Creating a cluster

The Crunchy Data stack does not automatically install a cluster like the Zalando stack does. Instead we must use the cli
to create a cluster.

1. `pgo create cluster gm-cluster --ccp-image=crunchy-postgres-gis --replica-count=2 --service-type=NodePort --metrics --autofail`

## References

* https://docs.geodan.io/ahpv2/howto_deployment/
* https://postgres-operator.readthedocs.io/en/latest/
* https://www.postgresql.eu/events/fosdem2018/sessions/session/1735/slides/59/FOSDEM%202018_%20Blue_Elephant_On_Demand.pdf
* http://sarahconway.com/slides/postgres-operator.pdf
* http://sarahconway.com/slides/postgres-operator.pdf
* https://www.postgresql-sessions.org/_media/10/postgresql_and_kubernetes_light.pdf
* https://medium.com/@Oskarr3/setting-up-ingress-on-minikube-6ae825e98f82
* https://pgdash.io/blog/postgres-11-sharding.html
* https://github.com/bitnami/bitnami-docker-pgpool

### Videos
* https://www.youtube.com/watch?v=G8MnpkbhClc
* https://www.youtube.com/watch?v=90kZRyPcRZw
