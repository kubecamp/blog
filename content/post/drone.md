+++
date = "2017-05-26T21:08:07+01:00"
title = "Running drone.io in Kubernetes"

+++

[Drone](http://readme.drone.io/) is a Continuous Integration platform built on Docker, written in Go. Every build is executed inside an ephemeral Docker container, giving developers complete control over their build environment with guaranteed isolation.

Configuring Drone to run in Kubernetes is fairly easy, but there are two things to take in consideration: which database to use and how to handle SSL. For the database you have a few choices: SQLite (default), MySQL and Postgres.

There are a few things you have to setup before starting the drone installation: DNS, GitHub OAuth Application and Database (if you want to use RDS or Google's Cloud SQL or a remote Posgres/MySQL).

Let's look at the configuration. First thing, we need to do is to create an `OAuth Application` in Github, you will have to give the app URL and the authorisation callback url. Since we are going to use an ingress controller, the DNS of our domain (in our case is `build.kube.camp`) points to the Ingress Controller service (`type: LoadBalancer`).

We're going to install Drone in Kubernetes using a [Helm](https://github.com/kubernetes/helm) chart. The Drone chart is fairly complex. It contains the following templates:

* Drone server deployment
* Drone agent deployment 
* Internal service for accessing the Drone server
* Ingress controller
* Drone secret (github client id, secret, etc).
* Drone-tls secret (SSL certificates for the ingress rule)
* Configmap (ngix conf to define headers for the server and the websockets when terminating the SSL in the Ingress controller or the LB).


Once you have DNS and Github details, we need to install the Database. We're going use Postgres as our database, and it will run in our cluster as well.

### Install Dependencies

We need to have installed in our cluster an [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress/) and a Postgres database using the charts availables in the charts repo: [Postgres Chart](https://github.com/kubernetes/charts/tree/master/stable/postgresql), [Ingress Chart](https://github.com/kubernetes/charts/tree/master/stable/nginx-ingress), please, refer to these charts to understand how both components will be deployed and how they need to be configured.

```bash
-> % helm repo update
-> % helm upgrade --install access stable/nginx-ingress --debug
-> % helm upgrade --install data stable/postgresql --debug --set postgresPassword=P1P4BTqd2XLw,postgresDatabase=drone,postgresUser=droneadmin

```

Once we have the DNS, database, GitHub details and, certificates, it's time to create our drone installation. We need to define a few ENVVARS with the secrets.

The Postgres `datasource` is formed as follows: 

```bash
postgres://root:pa55word@127.0.0.1:5432/postgres?sslmode=disable
```

this is 

```bash
postgres://USER:PASSWORD@HOST:5432/DATABASE?sslmode=disable
```

Note that the Postgres host is composed by the Postgres Helm `release` plus the Postgres chart name. Since our release was called `data` and the chart is called `postgres`, the host will be `data-postgres`


```bash
export GITHUB_CLIENT=xxx
export GITHUB_SECRET=xxx
export DATASOURCE=postgres://droneadmin:P1P4BTqd2XLw@data-postgres:5432/drone?sslmode=disable
export TLS_CERT=xxx
export TLS_KEY=xxx
```



### Install Drone

Once we have all the settings defined, we can execute Helm to install our release:

```bash
helm upgrade --install build . --debug --set github.secret=$GITHUB_SECRET,github.client=$GITHUB_CLIENT,shared_secret=ipsofactoreboltoso,database.datasource=$DATASOURCE,tls.crt=$TLS_CERT,tls.key=$TLS_KEY
```

Once the ingress rule is applied, we will be able to access Drone using the DNS that we have previously defined. In our case, `build.kube.camp`. Note that the ingress rule has a setting: 

```bash
ingress.kubernetes.io/ssl-redirect: "true"
```

that makes that all the `http` request, be redirected to `https`, this means that if we do `https://build.kube.camp`, we should see Drone.

### Drone builds

Kubernetes doesn't run the latest version of Docker, which it might give you a few headaches. If your builds fail because there's a mismatch in the Docker API version, the solution is fairly simple. If you look at how we build this blog (yes, it runs in kubernetes nd yes, it's built using a Drone that runs in Kubernetes as well):

```
docker-web:
    environment:
          - DOCKER_API_VERSION=1.24
    image: plugins/docker:1.12
    repo: quay.io/ipedrazas/kubecamp-blog
    tags: 
      - latest
      - ${DRONE_BRANCH}-${DRONE_COMMIT_SHA:0:7}
    registry: quay.io
    email: "info@info.com"
    debug: true
    secrets: [ docker_username, docker_password ]
```

As you can see, we specify the environment variable `DOCKER_API_VERSION=1.24`, another strategy is to run the drone agents with a sidekick container that runs the version of Docker that you want need.

__Resilient__
Now that we have Drone installed, let's do a few tests. __IF__ you're using a remote database you can destroy the `drone-server` pod safely. The pod will be re-created, and the server will re-connect to the database. 