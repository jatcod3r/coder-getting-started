# Deploying Coder on AWS

## Prerequisites

Make sure you have the following prepared:

- The K8s Cluster
- The K8s CLI (`kubectl`)
- The Helm CLI (`helm`)

Additionally, we'll need the following Helm Chart repositories installed:

- [Coder](https://helm.coder.com/v2)

```bash
$ helm repo add coder-v2 https://helm.coder.com/v2
```

- **(Optional)** [Bitnami](https://charts.bitnami.com/bitnami)

```bash
$ helm repo add bitnami https://charts.bitnami.com/bitnami
```


Within the cluster, make sure that you have the following configured:

- [Default Storage Class](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/)

## Step 1. Creating the Namespace(s)

To avoid grouping Coder with other non-Coder resources, we use [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/). We'll create one called `coder`.

```bash
$ kubectl create namespace coder
```

---

**(Optional)** If you don't have an external database, then we can configure an internal one. Make sure you also have a separate namespace for this. We'll call this `database`:

```bash
$ kubectl create namespace database
```

## Step 2. Preparing Your Helm Chart(s)

With our namespace(s) prepared, we can start deploying to them.

---

**(Optional)** If you don't have an external database, then can get one configured with a `database.yaml` for the Bitnami chart:

```yaml
auth:
  username: coder
  password: coder
  database: coder
persistence:
  size: 10Gi
```

This configuration defines the size, name, and credentials of a PostgreSQL Database.

For a full reference of Bitnami Helm values, visit their ["helm/bitnami"](https://artifacthub.io/packages/helm/bitnami/postgresql?modal=values) page.

---

For Coder, we create a `coder.yaml` file that contains:

```yaml
coder:
  env:
    # External Database (outside Cluster)
    - name: CODER_PG_CONNECTION_URL
      value: postgres://<username>:<password>@<postgresql-endpoint>:5432/<database>
  service:
    externalTrafficPolicy: Local
    sessionAffinity: None
    annotations: 
      service.beta.kubernetes.io/aws-load-balancer-internal: "false"
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

Keep in mind that this assumes your database isn't outside the cluster. If it's internal, then the YAML slightly changes `CODER_PG_CONNECTION_URL`

```yaml
coder:
  env:
    # Internal Database (in Cluster)
    - name: CODER_PG_CONNECTION_URL
      value: postgres://<username>:<password>@<service-name>.<namespace>.svc.cluster.local:5432/<database>?sslmode=disable
  service:
    externalTrafficPolicy: Local
    sessionAffinity: None
    annotations: 
      service.beta.kubernetes.io/aws-load-balancer-internal: "false"
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

We'll collect the `service-name` and `namespace` in the next step.

Overall, this is all you need for a Coder deployment with K8s. Additionally, make sure to replace `username`, `password`, and `database` with the fields used for your database.

For a full reference of Coder Helm values, visit our ["helm/coder-v2"](https://artifacthub.io/packages/helm/coder-v2/coder?modal=values) page. 

You can also check out our [`coder server`](https://coder.com/docs/reference/cli/server) document for a list of environment variables you can pass to Coder.

## Step 3. Deploying Your Helm Chart(s)

**(Optional)** If you don't have an external database, then deploy this first. Use the Helm CLI like so:

```bash
$ helm install coder-db bitnami/postgresql -n database -f database.yaml
```

This should return an output similar to the one below:

```bash
NAME: coder-db
LAST DEPLOYED: Tue Apr  1 12:42:37 2025
NAMESPACE: database
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 16.6.0
APP VERSION: 17.4.0
```

You'll need to verify if this had completed. Check if the PostgreSQL pod is ready by running:

```bash
$ kubectl get pods -n database
``` 

This should reflect a pod similar to `coder-db-postgresql-0` which has a `1/1` ready state and a status set to `Running`

```bash
NAME                    READY   STATUS    RESTARTS   AGE
coder-db-postgresql-0   1/1     Running   0          5s
```

Additionally, you should see if the database's [`Service`](https://kubernetes.io/docs/concepts/services-networking/service/) is available:

```bash
$ kubectl get svc -n database
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
coder-db-postgresql      ClusterIP   10.100.106.106   <none>        5432/TCP   5s
```

Now, going back to `Step 2. Preparing the Helm Chart`, make sure that the `coder.yaml` looks like:

```yaml
coder:
  env:
    - name: CODER_PG_CONNECTION_URL
      value: postgres://coder:coder@coder-db-postgresql.database.svc.cluster.local:5432/coder?sslmode=disable
  service:
    externalTrafficPolicy: Local
    sessionAffinity: None
    annotations: 
      service.beta.kubernetes.io/aws-load-balancer-internal: "false"
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

where `CODER_PG_CONNECTION_URL` references the database endpoint in a different namespace via the it's `Service`. 

---

To deploy Coder, you'll need to run:

```bash
$ helm install coder coder-v2/coder -n coder -f coder.yaml
```

Similar to the database output, this will return:

```bash
NAME: coder
LAST DEPLOYED: Tue Apr  1 14:38:49 2025
NAMESPACE: coder
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Enjoy Coder! Please create an issue at https://github.com/coder/coder if you run
into any problems! :)
```

To verify that the Coder deployment was successful, check if the Coder pods are available, as well as the service.

```bash
$ kubectl get pods -n coder
NAME                     READY   STATUS    RESTARTS   AGE
coder-79cfd77657-fk597   1/1     Running   0          44m

$ kubectl get service -n coder
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP                                             PORT(S)        AGE
coder   LoadBalancer   10.100.134.47   k8s-coder-coder-****-****.elb.us-east-1.amazonaws.com   80:32071/TCP   65m
```

## Troubleshooting

- 
- Error: INSTALLATION FAILED: create: failed to create: namespaces "..." not found
- [warn]  ping postgres: retrying  error="pq: SSL is not enabled on the server"
- Failed build model due to unable to resolve at least one subnet (0 match VPC and tags: [kubernetes.io/role/internal-elb]) 