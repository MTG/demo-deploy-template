# MTG kubernetes configuration template

This is a [cookiecutter](https://github.com/cookiecutter/cookiecutter) configuration for setting
up a service on the DTIC kubernetes cluster.

It should contain everything that you need to start with deploying an application on the DTIC kubernetes (k8s) infrastructure.

## Preliminaries

### Docker basics

This guide assumes that you already know how to build your application using Docker. At the very least, you should be able
to build some images and run them using docker-compose.

### Accounts and setup

You need two different accounts:

* An account on *Rancher*
* A regular DTIC account, with access to the rancher bastion host

Read the [DTIC documentation](https://docs.google.com/document/d/1asHiQGk75OdoRc2wLpPexZonnIjz3gIDN7eDvRF3rPc/edit) first.

See the section "Bastion server" in the documentation to see how to copy the kubectl credentials file to the k8s bastion host
or to your local machine. We recommend that you copy it to `~/.kube/config` so that you don't have to set any environment
variables before using `kubectl`

You can run `kubectl` either on the bastion host or locally on your machine. On macOS, install Docker desktop or use `brew install kubectl`
Otherwise, see the [kubernetes tools documentation](https://kubernetes.io/docs/tasks/tools/) to learn how to install it.

### Docker images

There is a private registry server available at `registry.sb.upf.edu`

Log in with

    docker login registry.sb.upf.edu

Use your DTIC username/password.

Tag your images something like

    registry.sb.upf.edu/MTG/your-application:version

## Generate configuration

The rest of this document assumes a basic web application with the following components

* A web application that does something
* Nginx in front of the web application for routing (web server)
* An NFS disk space for saving files
* Redis for caching data

Run cookiecutter to generate the configuration, answering the questions. This data can be changed at a later stage by
editing the generated files.

    # Install if needed
    pip install cookiecutter

    cookiecutter `pwd`/template

    project_name [myproject]: coolapp
    namespace [mtg-coolapp]:
    docker_config [~/.docker/config.json]:
    internal_hostname [coolapp.mtg.sb.upf.edu]:
    nfs_volume [/MTG/MTG-coolapp]:
    app_image [registry.sb.upf.edu/mtg/coolapp:1]:

See that once you choose a name, cookiecutter suggests some sane defaults that might be useful.

## Basics

### Namespaces

A project in k8s is isolated in a _namespace_. You will need to create a namespace for your project, normally `mtg-projectname`.
A namespace has _resource limits_, which describe the maximum number of CPU cores and amount of memory that the project can use.
You will need to estimate how many resources you will need for all of the containers in your project.

You can make a namespace from Rancher. Open the Cluster-Projects/Namespaces item in the left menu, then click
"Create Namespace"

### Image pull secrets

Rancher needs to be authenticated to the DTIC registry server in order to be able to pull images from it.
See the section "Create Rancher Secret" in the DTIC rancher guidelines. Name your secret `projectname-registry`

TODO: Use docker config on a local machine to generate a secret.

### Images

Remember that docker hub has [pull limits](https://docs.docker.com/docker-hub/download-rate-limit/) when pulling without authentication.
For this reason, we recommend keeping a copy of all public images in the DTIC registry server (e.g. nginx) to prevent problems during deploy.
See the file `images/Makefile` for a basic Makefile which pulls an image from docker hub, tags it to the DTIC registry server, and pushes.
If you want to upgrade a version of one of your public images, update this file.

## Gateways and Virtual services

These are items needed by the kubernetes infrastructure to route traffic from a domain name to your application.

Gateway (`gws` directory): The cookiecutter scripts automatically create a gateway with the same name as the namespace.
Each namespace needs a gateway.

Virtual services (`vs` directory): maps a DNS name into the namespace. There is magic routing configured in the UPF
network for `*.mtg.sb.upf.edu`. If you specify a subdomain in your virtual service then you will immediately be able to
visit this host using https.
This domain is internal to the UPF network, so you can only access it from on campus (wired network) or via
the VPN (if on eduroam or accessing remote).

for example:

```yaml
spec:
  hosts:
  - coolapp.mtg.sb.upf.edu
  - coolapp.upf.edu
```

will make https://coolapp.mtg.sb.upf.edu available (note that this is available on HTTPS, even though your service
is probably published on HTTP)

When we want to publish the application on a publicly available address, we ask for IT to route traffic from the
gateway to this domain. You also need to add the public domain name to the `hosts` list so that it can route it.
For example, open a CAU (Reserca->Hostatjament) requesting something like the following:

> Hi,
> I have a new application running on kubernetes. Can you please configure the public address
> https://coolapp.upf.edu to redirect to the internal hostname https://coolapp.mtg.sb.upf.edu?
> Thanks!

A virtual service routes traffic to a defined _Service_ (see below). Set the `host` to the name of the service
and `port` to the published port. If you have multiple domains that you want to access the same container, you can
have one VirtualService with multiple `hosts`. If you want many containers to be accessible via different hosts,
you should configure multiple VirtualServices each with a specific host name and a specific destination.
If you want two different suburls, e.g. https://coolapp.mtg.sb.upf.edu/one and https://coolapp.mtg.sb.upf.edu/two,
you can either specify two separate `spec.http.match` items with different `uri.prefix` values, or send all traffic
to an nginx instance and route to different services there.

```yaml
spec:
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: **nginx**
        port:
          number: 80
      weight: 100
```

TODO: Virtual services can do a lot of complex routing/rate limiting/etc. Add some common examples here.

## Application configuration

You can add dynamic configuration into your running container by using ConfigMaps. These can be used to either add environment
variables, or load an entire file into your container (nginx config, django settings, etc).

See more information about ConfigMaps at https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/

If you have an `environment`-style file, such as

```
PORT=8080
MYSQL_HOST=192.168.1.100
MYSQL_DATABASE=my_db
```

Then you can create a ConfigMap directly and apply it to the cluster

    kubectl create configmap app-configs --from-env-file=application.properties

If you want to just generate a kubernetes configuration file and not apply it to the cluster, you can use

    kubectl create configmap app-configs --from-env-file=application.properties --dry-run=client -o yaml > app-configs.yaml

If you want to specify each environment variable on the command line you can do so, but this might become unwieldy

You can also create a ConfigMap that contains an entire file (or more than one). In this template example, we use this
for the nginx config file and sample django local_settings.py.

Create and apply:

    kubectl create configmap app-django-config --from-file=local_settings.py

Create as a config file:

    kubectl create configmap app-django-config --from-file=local_settings.py --dry-run=client -o yaml > app-django-config.yaml

This creates a ConfigMap called `app-django-config` with a sub-key `local_settings.py`. You can add multiple `--from-file` items to add
many files to the ConfigMap. If you want to change the name of the subkey, use `--from-file=newname=filename.py`. Apply these
files to your Deployment using a Volume configuration.


## Deployments and Services

Deployments: this is like a "service" in docker-compose. It includes the image name, resource limits,
environment variables, and volumes (normally from external NFS mounts).

### Environment variables

To apply [all items in a ConfigMap as environment variables](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#configure-all-key-value-pairs-in-a-configmap-as-container-environment-variables),
use this in your deployment file:

```yaml
spec:
  containers:
    - name: test-container
      envFrom:
      - configMapRef:
          name: coolapp-web-env
```

### Services

Any deployment which accepts network traffic has to have an explicit service which states its name and what port it's available on.
This is the case for both external deployments (e.g. nginx) and internal deployments (e.g. a database or redis)

Services become available automatically at `servicename.namespace.svc.cluster.local`, or `service.namespace` or `service`

When you make a VirtualService, this redirects traffic to a kubernetes Service. Kubernetes uses
[selectors, labels, and names](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) to route traffic.

Selectors and labels between a VirtualService, Service, and Deployment work like this
![Selectors and labels between a VirtualService, Service, and Deployment](/kubernetes/service-labels.png)

### Volumes

If you need to store permant data (assets, models, uploads, databases) then you need to store it on an NFS volume.

Request space for your application by opening a CAU (Recerca->Emmagatzematge)

> Hi, I require an NFS volume for a new MTG project, coolapp.
> This volume should be accessible via the kubernetes cluster. The following users need to be able to access it: [list of users]

They will get back to you with the name of the NFS mount, it'll likely be called MTG-coolapp

In order to define an NFS storage mount in a Deployment, first create a `volume` in the deployment file:

```yaml
      volumes:
        - name: coolapp-static
          nfs:
            path: /MTG/MTG-coolapp
            server: 10.80.110.228
```

Then you can access this volume from the pod definition:
```yaml
spec:
  template:
    ...
    spec:
      ...
      containers:
        - ...
          volumeMounts:
            - mountPath: /static
              name: coolapp-static
```

Note that we don't use PersistentVolumes in our setup to make it easier for users to manage deployments without requiring the IT
department to perform maintenance on them for us.

If you need to access this NFS space manually to upload content to, set up an SFTP server. See the [SFTP docs](../sftp) for more
information about this. If you requested that specific DTIC user accounts have access to this space, the IT department will
create a user group for you to use in the SFTP configuration.

We recommend creating at least one subfolder in your volume so that you can manage file permissions for access both via
k8s containers and SFTP:

    10.80.110.228/MTG/MTG-coolapp/data

If you need to make multiple subfolders of your NFS volume available in a container at different locations, you don't need to make
multiple `volumes` definitions. Instead, make one and use `subPath` inside `volumeMounts`:

```yaml
          volumeMounts:
            - mountPath: /static
              name: coolapp-static
              subPath: static
            - mountPath: /data
              name: coolapp-static
              subPath: data
```

To mount a ConfigMap file into a container, define it as a volume:

```yaml
      volumes:
        - name: django-config
          configMap:
            name: app-django-config
```

And then attach it to a location using volumeMounts and subPath:

```yaml
          volumeMounts:
            - mountPath: /code/local_settings.py
              name: app-django-config
              subPath: local_settings.py
```

TODO: You can mount a ConfigMap as a directory, and all files in the ConfigMap will appear there. If you update the ConfigMap, then the contents of the
files in the running container will change without needing to restart it.

## Secrets/passwords

You can store config items as a _secret_, which will be stored encrypted in kubernetes. It works in much the same way
as a ConfigMap.

## Readiness/liveness probes

TODO: These can be used for k8s to determine if your container is ready to receive traffic or not.

## Stateful sets

Use the type StatefulSet instead of a Deployment for a service which should only have one instance running (e.g. a database server)

## Applying all of your files

* Which order to apply all items

## Useful features in rancher

* Debugging
* Console
* Logs
* Checking deployment state

## Some kubectl commands

    kubectl -n mtg-fsdatasets apply -f deployments/web.yaml

    kubectl -n mtg-trompa rollout restart deployment/nss

    kubectl -n mtg-fsdatasets get pods
    kubectl -n mtg-fsdatasets describe pod web

    kubectl -n mtg-webs delete deployment/web


## Releasing new versions
- ideas about using image versions
- different types of rollouts
  * limits (cannot create new and then delete old)
  * Destroy vs recreate deploy method

## Accessing files (sftp)

