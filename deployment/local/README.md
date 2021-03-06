# Run a local OpenWPM crawl using Kubernetes

Documentation and scripts to launch an OpenWPM crawl on a Kubernetes cluster locally.

## Prerequisites

Install Docker and Kubernetes locally. Note that
[Docker for Mac](https://docs.docker.com/docker-for-mac/install/) includes
[Kubernetes](https://docs.docker.com/docker-for-mac/#kubernetes). Depending on
your platform you may also need to install a local cluster (.e.g,
[Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/).

For the remainder of these instructions, you are assumed to be in the `deployment/local/` folder.

## Ensure cluster is running

```
$ kubectl cluster-info
Kubernetes master is running at https://...
KubeDNS is running at https://...
```

If a cluster isn't running and you're using Minikube as your local cluster,
you can start it by running `minikube start --cpus 4 --memory 4096` (you can
change the number of CPUs and memory as necessary).

## Build Docker image

Make sure that you have an up to date docker image locally:

```
docker pull openwpm/openwpm
docker tag openwpm/openwpm openwpm
```

Alternatively, you can build the image from a local OpenWPM code repository:

```
cd path/to/OpenWPM
docker build -t openwpm .
cd -
```

If you're using Minikube as your local cluster, you'll need to set the Docker
environment prior to building the local container to avoid `ErrImageNeverPull`:

```
cd path/to/OpenWPM
eval $(minikube docker-env)
docker build -t openwpm .
cd -
```

## Set up a mock S3 service

```
kubectl apply -f localstack.yaml
```

## Deploy the redis server which we use for the work queue

```
kubectl apply -f redis.yaml
```

## Adding sites to be crawled to the queue

Create a comma-separated site list as per:

```
echo "1,http://www.example.com
2,http://www.example.org
3,http://www.princeton.edu
4,http://citp.princeton.edu/" > site_list.csv

../load_site_list_into_redis.sh crawl-queue site_list.csv 
```

(Optional) To load Alexa Top 1M into redis:

```
cd ..; ./load_alexa_top_1m_site_list_into_redis.sh crawl-queue; cd -
```

You can also specify a max rank to load into the queue. For example, to add the
top 1000 sites from the Alexa Top 1M list:

```
cd ..; ./load_alexa_top_1m_site_list_into_redis.sh crawl-queue 1000; cd -
```

(Optional) Use some of the `../../utilities/crawl_utils.py` code. For instance, to fetch and store a sample of Alexa Top 1M to `/tmp/sampled_sites.json`:
```
source ../../venv/bin/activate
cd ../../; python -m utilities.get_sampled_sites; cd -
```

## Deploying the crawl Job

Since each crawl is unique, you need to configure your `crawl.yaml` deployment configuration. We have provided a template to start from:
```
cp crawl.tmpl.yaml crawl.yaml
```

- Update `crawl.yaml`. This may include:
    - spec.parallelism
    - spec.containers.image
    - spec.containers.env

When you are ready, deploy the crawl:

```
kubectl create -f crawl.yaml
```

Note that for the remainder of these instructions, `metadata.name` is assumed to be set to `local-crawl`.

### Monitor Job

#### Queue status

Open a temporary instance and launch redis-cli:
```
kubectl exec -it redis-box -- sh -c "redis-cli -h localhost"
```

Current length of the queue:
```
llen crawl-queue
```

Amount of queue items marked as processing:
```
llen crawl-queue:processing 
```

Contents of the queue:
```
lrange crawl-queue 0 -1
lrange crawl-queue:processing 0 -1
```

#### Using the Redis Commander UI

(Optional) You can alternatively use [a web UI](https://github.com/joeferner/redis-commander) to inspect the contents of the queue:
```
kubectl apply -f https://raw.githubusercontent.com/motin/redis-commander/issue-360/k8s/redis-commander/deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/motin/redis-commander/issue-360/k8s/redis-commander/service.yaml
kubectl port-forward svc/redis-commander 8081:8081
open http://localhost:8081/
```

#### Job status

```
watch kubectl get pods --selector=job-name=local-crawl
```

(Optional) To see a more detailed summary of the job as it executes or after it has finished:

```
kubectl describe job local-crawl
```

(Optional) If you're using the Minikube cluster, you can view the cluster and
job status by running:

```
minikube dashboard
```

#### View Job logs

```
mkdir -p local-crawl-results/logs
for POD in $(kubectl get pods --selector=job-name=local-crawl | grep -v NAME | grep -v Terminating | awk '{ print $1 }')
do
    kubectl logs $POD > local-crawl-results/logs/$POD.log
done
```

The crawl logs will end up in `./local-crawl-results/logs`

#### Using the Kubernetes Dashboard UI

(Optional) You can also spin up the Kubernetes Dashboard UI as per [these instructions](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/#deploying-the-dashboard-ui) which will allow for easy access to status and logs related to running jobs/crawls.

### Inspecting crawl results

When it has completed, run:
```
s3cmd --verbose --access_key=foo --secret_key=foo --host=http://localhost:32001 --host-bucket=localhost --no-ssl sync --delete-removed s3://localstack-foo local-crawl-results/data
```

The crawl data will end up in Parquet format in `./local-crawl-results/data`

### Clean up created pods, services and local artifacts

```
mkdir /tmp/empty
s3cmd --verbose --access_key=foo --secret_key=foo --host=http://localhost:32001 --host-bucket=localhost --no-ssl sync --delete-removed --force /tmp/empty/ s3://localstack-foo
kubectl delete -f crawl.yaml
kubectl delete -f localstack.yaml
kubectl delete -f redis.yaml
kubectl delete pod temp
rm -r local-crawl-results/data
rm -r local-crawl-results/logs
```
