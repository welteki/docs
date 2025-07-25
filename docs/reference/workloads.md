# Workloads

OpenFaaS can host multiple types of workloads from functions to microservices, but FaaS Functions have the best support.

## Common properties

All workloads must:

* serve HTTP traffic on TCP port 8080
* assume ephemeral storage
* be stateless
* have a name of no more than 63 characters

And integrate with a health-check mechanism with Kubernetes:

* implement `/_/health` as a HTTP endpoint
* implement `/_/ready` as a HTTP endpoint

The path for the HTTP ready and health endpoints can be overridden with annotations (see below)

If running in read-only mode, then you can write files to the `/tmp/` mount only. These files may be accessible upon subsequent requests but it is not guaranteed. For instance - if you have two replicas of a function then both may have different contents in their `/tmp/` mount. When running without read-only mode you can write files to the user's home directory subject to the same rules.

### Functions

To build a function simply use the [OpenFaaS CLI](/cli/install) to scaffold a new function using one of the official templates or one of your own templates. All FaaS Functions make use of the [OpenFaaS classic watchdog](/architecture/watchdog) or the next-gen [of-watchdog](https://github.com/openfaas/of-watchdog).

```
faas-cli template pull
faas-cli new --list
```

Or build your own templates Git repository and then pass that as an argument to `faas-cli template pull`

```
faas-cli template pull https://github.com/my-org/templates
faas-cli new --list
```

Custom binaries can also be used as a function. Just use the `dockerfile` language template and replace the `fprocess` variable with the command you want to run per request. If you would like to pipe arguments to a CLI utility you can prefix the command with `xargs`.

### Running an existing Docker image on OpenFaaS

If you can change the code in your application, you'll need to add a health and readiness endpoint, along with changing its HTTP port to listen on 8080, then you can deploy it directly to OpenFaaS.

Let's assume you cannot change any code, and have a Node.js application that listens to traffic on port 3000. We can use the OpenFaaS of-watchdog in HTTP mode to proxy traffic to the process and to provide health checks. 

You can view its Dockerfile and code at: [alexellis/expressjs-k8s](https://github.com/alexellis/expressjs-k8s) and the image is published to the Docker Hub at: `alexellis2/service:0.3.6`

Start by creating a new folder:

```bash
mkdir -p node-service/
```

Write a custom Dockerfile `./node-service/Dockerfile`:

```Dockerfile
# Import the OpenFaaS of-watchdog
FROM ghcr.io/openfaas/of-watchdog:0.9.16 as watchdog

# Add a FROM line from your existing image
FROM alexellis2/service:0.4.1

# Let's say that the image listens on port 3000 and 
# that we can't change that easily
ENV http_port 3000

# Install the watchdog from the base image
COPY --from=watchdog /fwatchdog /usr/bin/

# Now set the watchdog as the start-up process
# Along with the HTTP mode, and an upstream URL to 
# where your HTTP server will be running from the original
# image.
ENV mode="http"
ENV upstream_url="http://127.0.0.1:3000"

# Set fprocess to the value you have in your base image
ENV fprocess="node index.js"
CMD ["fwatchdog"]
```

Now create a stack.yml at the root directory `./stack.yml`:

```yaml
provider:
  name: openfaas
functions:
  node-service:
    handler: ./node-service
    image: docker.io/alexellis2/node-service-watchdog:0.1.0
    lang: dockerfile
```

Now run `faas-cli up`

Your code will now listen on port 8080 and implement the workload definition for OpenFaaS including automatic health-checks and a graceful shutdown.

You can then access the service at: `http://127.0.0.1:8080/function/node-service`

### Custom service account

!!! info "OpenFaaS Pro feature"
    This feature is part of the [OpenFaaS Pro](/openfaas-pro/introduction) distribution.

When using Kubernetes, OpenFaaS Pro functions can assume a ServiceAccount in the namespace in which they are deployed.

For example if a workload needed to read logs from the Kubernetes API using a ServiceAccount named `function-logs-sa`, you could bind it in this way:

*stack.yml*

```yaml
functions:
  system-logs:
     annotations:
       com.openfaas.serviceaccount: function-logs-sa
```

Here is an example `Role` that can list pods and work with Pod logs within the `openfaas-fn` namespace:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: function-logs-role
  namespace: openfaas-fn
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "create"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: function-logs-role-binding
  namespace: openfaas-fn
subjects:
- kind: ServiceAccount
  name: function-logs-sa
  namespace: openfaas-fn
roleRef:
  kind: Role
  name: function-logs-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: function-logs-sa
  namespace: openfaas-fn
  labels:
    app: openfaas
```

For a full code example and additional use cases, read: [Learn how to access the Kubernetes API from a Function ](https://www.openfaas.com/blog/access-kubernetes-from-a-function/)

### Custom TerminationGracePeriod for long running functions

!!! info "OpenFaaS Pro feature"
    This feature is part of the [OpenFaaS Pro](/openfaas-pro/introduction) distribution.

You can configure your functions to drain any requests in flight when scaling down. This prevents errors and makes sure all work is processed, before Kubernetes finally removes any Pods.

To set a custom [TerminationGracePeriod](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/) for a function, configure a `write_timeout` environment variable.

```yaml
functions:
  email-batch:
     environment:
       write_timeout: "1m"
```

When scaling down the function after scaling up, or scaling to zero, Kubernetes will wait for 1m before removing the function. If there is no work to be done, it could exit sooner because the OpenFaaS watchdog does a safe shutdown.

Read more here: [Improving long-running jobs for OpenFaaS users](https://www.openfaas.com/blog/long-running-jobs/)

### Stateless microservices

A stateless microservice can be built using the `dockerfile` language type and the OpenFaaS CLI - or by building a custom Docker image which serves traffic on port `8080` and deploying that via the RESTful API, CLI or UI.

An example of a stateless microservice may be an Express.js application using Node.js, a Sinatra app with Ruby or an ASP.NET 2.0 Core website.

Use of the [OpenFaaS next-gen of-watchdog](https://github.com/openfaas/of-watchdog) is optional, but recommended for stateless microservices to provide a consistent experience for timeouts, logging and configuration.

On Kubernetes is possible to run any container image as an OpenFaaS function as long as your application exposes port 8080 and has a HTTP health check endpoint.

### Custom HTTP health checks

!!! info "OpenFaaS Pro feature"
    This feature is part of the [OpenFaaS Pro](/openfaas-pro/introduction) distribution.

Liveness and readiness probes can be set globally for the installation: [OpenFaaS chart reference](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#faas-netes--operator).

Annotations can be used to configure probes on a per function basis. Any overrides set in annotations will take precedence over the global configuration.

You can specify the HTTP path of your health check and control the behavior of the probe with the following annotations:

* `com.openfaas.health.http.path`
* `com.openfaas.health.http.initialDelaySeconds`
* `com.openfaas.health.http.periodSeconds`
* `com.openfaas.health.http.timeoutSeconds`
* `com.openfaas.health.http.failureThreshold`

Readiness probes use the same HTTP path as the health check by default. The path, and other probing fields can be configured with these annotations:

* `com.openfaas.ready.http.path`
* `com.openfaas.ready.http.initialDelaySeconds`
* `com.openfaas.ready.http.periodSeconds`
* `com.openfaas.ready.http.timeoutSeconds`
* `com.openfaas.ready.http.successThreshold`
* `com.openfaas.ready.http.failureThreshold`

For example, you may have a function that takes 30s to initialise, but then only needs to be checked every 5s after that.

```yaml
functions:
  kubesec:
    image: docker.io/stefanprodan/kubesec:v2.1
    skip_build: true
    annotations:
      com.openfaas.ready.http.path: "/_/ready"
      com.openfaas.ready.http.initialDelaySeconds: 30
      com.openfaas.ready.http.periodSeconds: 5
```

### How to find the function's name at runtime

!!! info "OpenFaaS Pro feature"
    This feature is part of the [OpenFaaS Pro](/openfaas-pro/introduction) distribution.

OpenFaaS exposes some information to functions through environment variables.

The function name is made available in every function as an environment variable `OPENFAAS_NAME`. You can test this by deploying the `env` function the store and invoking it, i.e.

```bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=env-648d57f457-xtvwg
OPENFAAS_NAME=env
fprocess=env
HOME=/home/app
Http_X_Forwarded_Proto=https
```

### Supported content types

OpenFaaS supports workloads over HTTP, and most standard content types are supported.

Since OpenFaaS has no hard limit on function execution duration, it allows for maintaining long-lived connections for streaming over HTTP.

> **Important**: Always ensure OpenFaaS system and function timeouts are configured appropriately for your streaming workloads. See [Extended timeouts](https://docs.openfaas.com/tutorials/expanded-timeouts/) for details.

Supported streaming options:

**Server-Sent Events (SSE)**

Server-Sent Events enable a function to push one-way event streams to a client.

- Clients should include an `Accept: text/event-stream` header in their request when starting tht SSE request.
- The function's response `Content-Type` header must be set to `text/event-stream`. Each event data chunk should be prefixed with data: and terminated by two newline characters (\n\n).

For more details on SSE, refer to [Using Server Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)

**WebSockets**

WebSockets provide bidirectional, full-duplex communication between a client and an OpenFaaS function.

To create a WebSocket-enabled function, you can modify an existing OpenFaaS template or use the `Dockerfile` template as a starting point.

For a comprehensive guide, check out: [How to Integrate WebSockets with Serverless Functions and OpenFaaS](https://www.openfaas.com/blog/serverless-websockets/)

**Newline Delimited JSON (NDJSON)**

NDJSON (or JSON Lines) is a format for streaming multiple independent JSON objects, each on a new line.

- Clients should include an `Accept: application/x-ndjson` header in their request.
- The function's response `Content-Type` header should be set to `application/x-ndjson`. Each line in the response should be a complete JSON object followed by a newline character (\n).
