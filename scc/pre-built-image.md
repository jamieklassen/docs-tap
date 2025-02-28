# Pre-built image

For applications that have a pre-defined way of building 
container images, the supply chains in the Out of the Box packages
support specifying a pre-built image used in the final application 
using the same stages as any other Workload.


## <a id="workload"></a> Workload

To select a pre-built image, the `workoad.spec.image` field must be set with
the name of the container image that contains the application to deploy.

Using the Tanzu CLI, run the `--image` field of `tanzu apps
workload create` command:

```console
$ tanzu apps workload create --help
--image image        pre-built image, skips the source resolution and build 
                     phases of the supply chain
```

For example, if you have an image named `IMAGE`, you can create a workload
with the flag mentioned earlier:

```bash
tanzu apps workload create tanzu-java-web-app \
  --app tanzu-java-web-app \
  --type web \
  --image IMAGE
```
```console
Create workload:
      1 + |---
      2 + |apiVersion: carto.run/v1alpha1
      3 + |kind: Workload
      4 + |metadata:
      5 + |  labels:
      6 + |    app.kubernetes.io/part-of: hello-world
      7 + |    apps.tanzu.vmware.com/workload-type: web
      8 + |  name: tanzu-java-web-app
      9 + |  namespace: default
     10 + |spec:
     11 + |  image: IMAGE
```

## <a id="NAME"></a> Out of the Box Supply Chains

In Tanzu Application Platform, the `ootb-supply-chain-basic`, `ootb-supply-chain-testing`, and
`ootb-supply-chain-testing-scanning` packages each receive a new
supply chain that provides a pre-built container image for your
application.

```
ootb-supply-chain-basic

    (cluster)  basic-image-to-url   ClusterSupplyChain            (!) new
    ^          source-to-url        ClusterSupplyChain


ootb-supply-chain-testing

    (cluster)  testing-image-to-url  ClusterSupplyChain           (!) new
    ^          source-test-to-url    ClusterSupplyChain


ootb-supply-chain-testing-scanning

    (cluster)  scanning-image-scan-to-url    ClusterSupplyChain   (!) new
    ^          source-test-scan-to-url       ClusterSupplyChain
```

To leverage the supply chains that expect a pre-built image, the only necessary change to
the Workload is `workoad.spec.image` field being set to the
name of the container image that brings the application to be deployed.

The new supply chains use a new Cartographer feature
that lets VMware increase the specificity of supply chain selection by using
the `matchFields` selector rule. 

The selection takes place as follows:

- _ootb-supply-chain-basic_
  - from source: label `apps.tanzu.vmware.com/workload-type: web`
  - pre-built image: label `apps.tanzu.vmware.com/workload-type: web` **and**
    `workload.spec.image` set

- _ootb-supply-chain-testing_
  - from source: labels `apps.tanzu.vmware.com/workload-type: web` and `apps.tanzu.vmware.com/has-tests: true`
  - pre-built image: label `apps.tanzu.vmware.com/workload-type: web` **and**
    `workload.spec.image` set

- _ootb-supply-chain-testing-scanning_
  - from source: labels `apps.tanzu.vmware.com/workload-type: web` and `apps.tanzu.vmware.com/has-tests: true`
  - pre-built image: label `apps.tanzu.vmware.com/workload-type: web` **and**
    `workload.spec.image` set

Workloads that already work with the supply chains before Tanzu Application Platform v1.01.00
continue to work with the same supply chain.
Workloads that bring a pre-built container image must set
`workload.spec.image`.



## <a id="NAME"></a> How it works

An `ImageRepository` object is created to keep track of
new images pushed under that name, making them available to further resources in the
supply chain the final digest of the latest that is found.

Whenever a new image is pushed with that image name, the `ImageRepository`
object is detected, and then it is available to further resources by
updating its `imagerepository.status.artifact.revision` with the absolute
name of that image.

For example, if you create a Workload using an image named `hello-world`,
tagged `tanzu-java-web-app` hosted under `ghcr.io` in the `kontinue`
repository:

```bash
tanzu apps workload create tanzu-java-web-app \
  --app tanzu-java-web-app \
  --type web \
  --image ghcr.io/kontinue/hello-world:tanzu-java-web-app
```

After a couple seconds, you see the `ImageRepository` object created for
keeping track of images named
`ghcr.io/kontinue/hello-world:tanzu-java-web-app`:


```scala
Workload/tanzu-java-web-app
├─ImageRepository/tanzu-java-web-app        
├─PodIntent/tanzu-java-web-app
├─ConfigMap/tanzu-java-web-app
└─Runnable/tanzu-java-web-app-config-writer
  └─TaskRun/tanzu-java-web-app-config-writer-p2lzv
    └─Pod/tanzu-java-web-app-config-writer-p2lzv-pod
```

Inspecting the `Workload` status (more specifically,
`workload.status.resources`), you see the `image-provider` resource
promoting to further resources the image it found:

```yaml
apiVersion: carto.run/v1alpha1
kind: Workload
spec:
  image: ghcr.io/kontinue/hello-world:tanzu-java-web-app
status:
  resources:
    - name: image-provider
      outputs:
        # output being made available to further resources in the supply chain
        # (in this case, the latest image it found under that name).
        #
        - name: image
          lastTransitionTime: "2022-04-01T15:05:01Z"
          preview: ghcr.io/kontinue/hello-world:tanzu-java-web-app@sha256:9fb930a...

      # reference to the object managed by the supply chain for this
      # resource
      #
      stampedRef:
        apiVersion: source.apps.tanzu.vmware.com/v1alpha1
        kind: ImageRepository
        name: tanzu-java-web-app
        namespace: workload
      
      # reference to the template that defined how this object should look
      # like
      #
      templateRef:
        apiVersion: carto.run/v1alpha1
        kind: ClusterImageTemplate
        name: image-provider-template
```

The image found by the ImageRepository object is carried through the
supply chain to the final configuration that is pushed to either
a Git repository or image registry so that it is deployed in a run cluster.

>**Note:** The image name matches the image name supplied in the `workload.spec.image` field, but also includes the digest of the latest image found under the tag. If a new image is pushed to the same tag, you see the `ImageRepository` resolving the name to a different digest corresponding to the new image pushed.

## <a id="examples"></a> Examples

Aside from the gotchas outlined in the following section, it's transparent
to the supply chain how you come up with the image provided.

In the following examples below, VMware showcases a couple ways that you can build
container images for a Java-based application and complete the
supply chains to a running service.


### <a id="dockerfile"></a> Dockerfile

Using a Dockerfile is the most common way of building container images. You
can select a base image, on top of which certain operations must occur, such as compiling code, mutating the contents of the file system
to a final container image that has our application built and any
required runtime dependencies.

Here you use the `maven` base image for compiling our application code, and
then the minimal distroless `java17-debian11` image for providing a JRE
that can run our built application. 

With the image built, you push it to a container image registry, and then
reference it in the Workload.

1. Create a Dockerfile that describes how to build our application and make it
   available as a container image:

    ```Dockerfile
    ARG BUILDER_IMAGE=maven
    ARG RUNTIME_IMAGE=gcr.io/distroless/java17-debian11


    FROM $BUILDER_IMAGE AS build

            ADD . .
            RUN unset MAVEN_CONFIG && ./mvnw clean package -B -DskipTests


    FROM $RUNTIME_IMAGE AS runtime

            COPY --from=build /target/demo-0.0.1-SNAPSHOT.jar /demo.jar
            CMD [ "/demo.jar" ]
    ```

2. Push the container image to a container image registry: 

    ```bash
    docker build -t IMAGE .
    docker push IMAGE
    ```

3. Create a workload:

    ```bash
    tanzu apps workload create tanzu-java-web-app \
      --type web \
      --app tanzu-java-web-app \
      --image IMAGE
    ```
    ```console
    Create workload:
          1 + |---
          2 + |apiVersion: carto.run/v1alpha1
          3 + |kind: Workload
          4 + |metadata:
          5 + |  labels:
          6 + |    app.kubernetes.io/part-of: hello-world
          7 + |    apps.tanzu.vmware.com/workload-type: web
          8 + |  name: tanzu-java-web-app
          9 + |  namespace: default
         10 + |spec:
         11 + |  image: IMAGE
    ```

4. Run the following workload:

    ```bash
    tanzu apps workload get tanzu-java-web-app
    ```
    ```console
    # tanzu-java-web-app: Ready
    ---
    lastTransitionTime: "2022-04-06T19:32:46Z"
    message: ""
    reason: Ready
    status: "True"
    type: Ready

    Workload pods
    NAME                                                   STATUS      RESTARTS   AGE
    tanzu-java-web-app-00001-deployment-7d7df5ccf5-k58rt   Running     0          32s
    tanzu-java-web-app-config-writer-xjmvw-pod             Succeeded   0          89s

    Workload Knative Services
    NAME                 READY   URL
    tanzu-java-web-app   Ready   http://tanzu-java-web-app.default.example.com
    ```


### <a id="sb-maven"></a> Sprint Boot's `build-image` Maven target

For those familiar with Spring Boot's `build-image` target, you can make use of
it for coming up with a container image that runs out application, and making
use of it must work with the Dockerfile.

For example, using the same sample repository as mentioned before
(https://github.com/sample-accelerators/tanzu-java-web-app), you can run from
the root of that repository the following command:

```bash
IMAGE=ghcr.io/kontinue/hello-world:tanzu-java-web-app
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=$IMAGE
```
```console
[INFO] Scanning for projects...
[INFO]
[INFO] --------------------------< com.example:demo >--------------------------
[INFO] Building demo 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
...
[INFO]
[INFO] Successfully built image 'ghcr.io/kontinue/hello-world:tanzu-java-web-app'
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  39.257 s
[INFO] Finished at: 2022-04-06T19:40:16Z
[INFO] ------------------------------------------------------------------------
```

With the image built, you can push it to the container image registry:

```bash
IMAGE=ghcr.io/kontinue/hello-world:tanzu-java-web-app
docker push $IMAGE
```
```console
The push refers to repository [ghcr.io/kontinue/hello-world]
1dc94a70dbaa: Preparing
...
9d6787a516e7: Pushed
tanzu-java-web-app: digest: sha256:7140722ea396af69fb3d0ad12e9b4419bc3e67d9c5d8a2f6a1421decc4828ace size: 4497
```

Having pushed the container image, you see the same results as in the
section earlier where you built the image with a `Dockerfile`.

To know more about building container images for Spring Boot application, make
sure you check out [Spring Boot with Docker][sboot-docker]

[sboot-docker]: https://spring.io/guides/gs/spring-boot-docker


## <a id="gotchas"></a> Gotchas

As the supply chains still aim at Knative as the runtime for the container
image provided, the application must adhere to Knative standards when it comes
to bringing the container up and serving traffic to it:

- **Listen on port 8080**

The Knative service gets created with the pod template spec being set to have
the container port set to 8080, so it's expected that regardless of how the
application's container image is built, to have a socket listening on 8080.

```
ports:
  - containerPort: 8080
    name: user-port
    protocol: TCP
```

- **Non-privileged user ID**

By default, the container initiated as part of the pod is run as user
1000.

```
securityContext:
  runAsUser: 1000
```

- **Arguments other than the image's default entrypoint**

Because there is no way of supplying arguments down to the Pod template specifications
that wis be used in the Knative service, in most cases the container
image can run based solely on the default entrypoint configured for it
(in the case of Dockerfiles, the combination of ENTRYPOINT and CMD).

In case extra configuration must
used (see `--env` flags in `tanzu apps workload create` or
`workload.spec.env`).

- **Credentials for pulling the container image at runtime**

The image provided is not relocated to an internal container image registry. Any
components that are touching the image must have the necessary credentials
for pulling it. In practice, that means attaching to the service accounts used
for both Workload and Deliverable a secret that contains the credentials to
pull that container image.

Similarly, if the image is hosted in a registry whose certificates are
signed by a private certificate authority, not only the components of the
supply chains and delivery must trust it, but also the Kubernetes nodes in the
run cluster.
