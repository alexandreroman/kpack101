# KPack 101

This project shows how to use [KPack](https://github.com/pivotal/kpack),
an open-source project by [Pivotal](https://pivotal.io), to leverage
[Cloud Native Buildpacks](https://buildpacks.io) on any Kubernetes cluster.

Using KPack, you can automatically build secure Docker images from your source code,
without having to write a `Dockerfile`. Moreover, KPack can rebase your Docker images
when updates are available.

This repository describes how to deploy KPack to your Kubernetes cluster, and how to use
it to create your first Docker image.

## Deploying KPack

Download the [latest KPack release](https://github.com/pivotal/kpack/releases):
you should have a file `release.yaml`.

Deploy KPack using `kubectl`:
```bash
$ kubectl apply -f release.yaml
namespace/kpack configured
customresourcedefinition.apiextensions.k8s.io/builds.build.pivotal.io created
customresourcedefinition.apiextensions.k8s.io/builders.build.pivotal.io created
clusterrole.rbac.authorization.k8s.io/kpack-admin created
clusterrolebinding.rbac.authorization.k8s.io/kpack-controller-admin created
deployment.apps/kpack-controller created
customresourcedefinition.apiextensions.k8s.io/images.build.pivotal.io created
serviceaccount/controller created
customresourcedefinition.apiextensions.k8s.io/sourceresolvers.build.pivotal.io created
```

Check that KPack is running:
```bash
$ kubectl -n kpack get pods
NAME                               READY   STATUS    RESTARTS   AGE
kpack-controller-dd4bb9c58-rj2m9   1/1     Running   0          11m
```

## Creating a Docker image from a Git repository using KPack

Create a secret for push access to your Docker registry. Let's create file
`dockerhub-creds.yml` with your [Docker Hub](https://hub.docker.com) credentials:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dockerhub-creds
  annotations:
    build.pivotal.io/docker: index.docker.io
type: kubernetes.io/basic-auth
stringData:
  username: <username>
  password: <password>
```

Create a secret for read access to your Git repository. Create file
`github-creds.yml` to set your GitHub credentials:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-creds
  annotations:
    build.pivotal.io/git: https://github.com
type: kubernetes.io/basic-auth
stringData:
  username: <username>
  password: <password>
```

Please note you need to use a
[GitHub access token](https://github.com/settings/tokens) as your password in case
you enabled 2-Factor Authentication with your account.

You need a service account using your Docker registry and your Git repository credentials:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kpack-service-account
secrets:
  - name: dockerhub-creds
  - name: github-creds
```

We'll build Docker images using a Cloud Foundry buildpack
(don't worry: you can deploy the resulting Docker image anywhere):
```yaml
apiVersion: build.pivotal.io/v1alpha1
kind: Builder
metadata:
  name: cnb-builder
spec:
  image: cloudfoundry/cnb:cflinuxfs3
```

Finally, create a configuration file for building a Docker image from your
Git repository:
```yaml
apiVersion: build.pivotal.io/v1alpha1
kind: Image
metadata:
  name: spring-on-k8s-image
spec:
  tag: alexandreroman/spring-on-k8s # Set your Docker image
  serviceAccount: kpack-service-account
  builderRef: cnb-builder
  cacheSize: "2Gi"
  source:
    git:
      url: https://github.com/alexandreroman/spring-on-k8s.git # Use your Git repo URL
      revision: master
  build:
    env:
      - name: BP_JAVA_VERSION
        value: 8.* # Java 11 is used by default
```

Deploy all files to your Kubernetes cluster:
```bash
$ kubectl apply -f dockerhub-creds.yml
$ kubectl apply -f github-creds.yml
$ kubectl apply -f cnb-builder.yml
$ kubectl apply -f kpack-service-account.yml
$ kubectl apply -f app-source.yml
```

## Using KPack

Monitor KPack build status:
```bash
$ kubectl get cnbbuilds
NAME                                IMAGE   SUCCEEDED
spring-on-k8s-image-build-1-kfgz8           Unknown
```

Status is `Unknown` while image is being built.

Wait a couple of minutes, and the status will be updated:
```bash
kubectl get cnbbuilds                                    
NAME                                IMAGE                                                                                                                  SUCCEEDED
spring-on-k8s-image-build-1-kfgz8   index.docker.io/alexandreroman/spring-on-k8s@sha256:6188498e07a6c4e6620fd33bf7c2842f76618ae6f05f07e4146f7cf1f8cfd624   True
```

Go to your Docker registry: a new image is now available!

Check that your image is ready:
```bash
$ docker run --rm -p 8080:8080/tcp \
    index.docker.io/alexandreroman/spring-on-k8s@sha256:6188498e07a6c4e6620fd33bf7c2842f76618ae6f05f07e4146f7cf1f8cfd624

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.5.RELEASE)

2019-08-22 16:49:29.261  WARN 1 --- [           main] pertySourceApplicationContextInitializer : Skipping 'cloud' property source addition because not in a cloud
2019-08-22 16:49:29.277  WARN 1 --- [           main] nfigurationApplicationContextInitializer : Skipping reconfiguration because not in a cloud
2019-08-22 16:49:29.300  INFO 1 --- [           main] i.pivotal.demos.springonk8s.Application  : Starting Application on 5f2335b83afb with PID 1 (/workspace/BOOT-INF/classes started by vcap in /workspace)
2019-08-22 16:49:29.306  INFO 1 --- [           main] i.pivotal.demos.springonk8s.Application  : No active profile set, falling back to default profiles: default
2019-08-22 16:49:33.134  INFO 1 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2019-08-22 16:49:34.676  INFO 1 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port(s): 8080
2019-08-22 16:49:34.683  INFO 1 --- [           main] i.pivotal.demos.springonk8s.Application  : Started Application in 6.237 seconds (JVM running for 7.322)
```

Now, update your Git repository, and monitor KPack activity:
```bash
kubectl get cnbbuildsNAME                                IMAGE                                                                                                                  SUCCEEDED
spring-on-k8s-image-build-1-kfgz8   index.docker.io/alexandreroman/spring-on-k8s@sha256:6188498e07a6c4e6620fd33bf7c2842f76618ae6f05f07e4146f7cf1f8cfd624   True
spring-on-k8s-image-build-2-cbd5f                                                                                                                          Unknown
```

A new image is being built!

Run this command again, and you should have a new image:
```bash
kubectl get cnbbuilds      
NAME                                IMAGE                                                                                                                  SUCCEEDED
spring-on-k8s-image-build-1-kfgz8   index.docker.io/alexandreroman/spring-on-k8s@sha256:6188498e07a6c4e6620fd33bf7c2842f76618ae6f05f07e4146f7cf1f8cfd624   True
spring-on-k8s-image-build-2-cbd5f   index.docker.io/alexandreroman/spring-on-k8s@sha256:9ed04eb2e25f7056ae268c8441032e16feaa82de8195ebb489142d02c381fb3d   True
```

## Contribute

Contributions are always welcome!

Feel free to open issues & send PR.

## License

Copyright &copy; 2019 [Pivotal Software, Inc](https://pivotal.io).

This project is licensed under the [Apache Software License version 2.0](https://www.apache.org/licenses/LICENSE-2.0).
