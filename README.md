<!-- omit in toc -->
# Deploying Redis Enterprise on Kubernetes

* [Quickstart Guide](#quickstart-guide)
  * [Prerequisites](#prerequisites)
  * [Installation](#installation)
  * [Installation on OpenShift](#installation-on-openshift)
* [Configuration](#configuration)
  * [RedisEnterpriseCluster custom resource](#redisenterprisecluster-custom-resource)
  * [Private Repositories](#private-repositories)
  * [Pull Secrets](#pull-secrets)
  * [Advanced Configuration](#advanced-configuration)
* [Connect to Redis Enterprise Software web console](#How-to-connect-to-Redis-Enterprise-Software-web-console?)
* [Upgrade](#upgrade)
* [Supported K8S Distributions](#supported-k8s-distributions)

This page describes how to deploy Redis Enterprise on Kubernetes using the Redis Enterprise Operator. The Redis Enterprise Operator supports two Custom Resource Definitions (CRDs):
* Redis Enterprise Cluster (REC): an API to create Redis Enterprise clusters. Note that only one cluster is supported per operator deployment.
* Redis Enterprise Database (REDB): an API to create Redis databases running on the Redis Enterprise cluster.
Note that the Redis Enterprise Operator is namespaced.
High level architecture and overview of the solution can be found [HERE](https://docs.redislabs.com/latest/platforms/kubernetes/).

## Quickstart Guide

### Prerequisites

- A Kubernetes cluster version of 1.11 or higher, with a minimum of 3 worker nodes.
- A Kubernetes client (kubectl) with a matching version. For OpenShift, an OpenShift client (oc).
- Access to DockerHub, RedHat Container Catalog or a private repository that can serve the required images.


The following are the images and tags for this release:
| Component | k8s | Openshift |
| --- | --- | --- |
| Redis Enterprise | `redislabs/redis:6.2.10-90` | `redislabs/redis:6.2.10-90.rhel7-openshift` |
| Operator | `redislabs/operator:6.2.10-4` | `redislabs/operator:6.2.10-4` |
| Services Rigger | `redislabs/k8s-controller:6.2.10-4` | `redislabs/k8s-controller:6.2.10-4` |
> * RedHat certified images are available on [Redhat Catalog](https://access.redhat.com/containers/#/product/71f6d1bb3408bd0d) </br>


### Installation
The "Basic" installation deploys the operator (from the current release) from DockerHub and default settings. Recommended for KOPS, GKE, AKS, EKS, Rancher, VMWare Tanzu.
This is the fastest way to get up and running with a new Redis Enterprise on Kubernetes.

1. Create a new namespace:
    > Note:
    For the purpose of this doc, we'll use the name "demo" for our cluster's namespace.

    ```bash
    kubectl create namespace demo
    ```

    Switch context to the newly created namespace:

    ```bash
    kubectl config set-context --current --namespace=demo
    ```

2. Deploy the operator bundle

   To deploy the default installation with `kubectl`, the following command will deploy a bundle of all the yaml declarations required for the operator:

    ```bash
    kubectl apply -f bundle.yaml
    ```

    Alternatively, to run each of the declarations of the bundle individually, run the following commands *instead* of the bundle:

    ```bash
    kubectl apply -f role.yaml
    kubectl apply -f role_binding.yaml
    kubectl apply -f service_account.yaml
    kubectl apply -f crds/v1/rec_crd.yaml
    kubectl apply -f crds/v1alpha1/redb_crd.yaml
    kubectl apply -f admission-service.yaml
    kubectl apply -f operator.yaml
    ```

    Run `kubectl get deployment` and verify redis-enterprise-operator deployment is running.

    A typical response may look like this:

    ```bash
    NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
    redis-enterprise-operator          1/1     1            1           2m
    ```

3. Redis Enterprise Cluster custom resource - `RedisEnterpriseCluster`

   Create a `RedisEnterpriseCluster`(REC) using the default configuration, which is suitable for development type deployments and works in typical scenarios. The full list of attributes supported through the Redis Enterprise Cluster (REC) API can be found [HERE](redis_enterprise_cluster_api.md). Some examples can be found in the examples folder. 

    ```bash
    kubectl apply -f examples/v1/rec.yaml
    ```

    > Note:
    The Operator can only manage one Redis Enterprise Cluster custom resource in a namespace. To deploy another Enterprise Clusters in the same Kubernetes cluster, deploy an Operator in an additional namespace for each additional Enterprise Cluster required. Note that each Enterprise Cluster can effectively host hundreds of Redis Database instances. Deploying multiple clusters is typically used for scenarios where complete operational isolation is required at the cluster level.
  
4. Run ```kubectl get rec``` and verify creation was successful. `rec` is a shortcut for RedisEnterpriseCluster. The cluster takes around 5-10 minutes to come up.
    A typical response may look like this:
    ```
    NAME  AGE
    rec   5m
    ```
    > Note: Once the cluster is up, the cluster GUI and API could be used to configure databases. It is recommended to use the K8s REDB API that is configured through the following steps. To configure the cluster using the cluster GUI/API, use the ui service created by the operator and the default credentials as set in a secret. The secret name is the same as the cluster name within the namespace.
5. Redis Enterprise Database (REDB) Admission Controller:
    The Admission Controlller is recommended for use. It uses the Redis Enterprise Cluster to dynamically validate that REDB resources as configured by the operator are valid.
    Steps to configure the Admission Controller:
   > **Note:** Redis Labs' Redis Enterprise Operator can also be installed through the [Gesher Admission Proxy](admission/GESHER.md)
    * Wait for the secret to be created:
        ```shell script
            kubectl get secret admission-tls
            NAME            TYPE     DATA   AGE
            admission-tls   Opaque   2      2m43s
       ```
    
    * Enable the Kubernetes webhook using the generated certificate
   
         * Save the certificate into a local environmental variable
         ```shell script
         CERT=`kubectl get secret admission-tls -o jsonpath='{.data.cert}'`
         ```
         * Create a patch file
         ```shell script
         sed 's/NAMESPACE_OF_SERVICE_ACCOUNT/demo/g' admission/webhook.yaml | kubectl create -f -
   
         cat > modified-webhook.yaml <<EOF
         webhooks:
         - name: redb.admission.redislabs
           clientConfig:
             caBundle: $CERT
           admissionReviewVersions: ["v1beta1"]
         EOF
         ```
         * Patch the validating webhook with the certificate (caBundle)
         ```shell script
         kubectl patch ValidatingWebhookConfiguration redb-admission --patch "$(cat modified-webhook.yaml)"
         ```
      
    > **Note:** If you're not using multiple namespaces you may skip to ["Verify the installation"](#verify_admission_installation) step.
   
    * Limiting the webhook to the relevant namespaces:    
      Unless limited, webhooks will intercept requests from all namespaces.<br>
      In case you have several REC objects on your K8S cluster you need to limit the webhook to the relevant namespace.  
      This is done by adding a `namespaceSelector` to the webhook spec that targets a label found on the namespace.<br>
    
         * First, make sure you have such a relevant label on the namespace and that it is unique for this namespace. e.g.
         
        ```yaml
        apiVersion: v1
        kind: Namespace
        metadata:
          labels:
            namespace-name: staging
        name: staging
        ```
        
        * Then, patch the webhook with a namespaceSelector. See this example:
        ```shell script
        cat > modified-webhook.yaml <<EOF
        webhooks:
        - name: redb.admission.redislabs
          namespaceSelector:
            matchLabels:
              namespace-name: staging
        EOF
        ```
        
        * apply the patch:
        ```shell script
        kubectl patch ValidatingWebhookConfiguration redb-admission --patch "$(cat modified-webhook.yaml)"
        ```
       <a name="verify_admission_installation"></a>
       * Verify the installation
        In order to verify that the all the components of the Admission Controller are installed correctly, we will try to apply an invalid resource that should force the admission controller to reject it.  If it applies succesfully, it means the admission controller has not been hooked up correctly.
        
        ```shell script
        $ kubectl apply -f - << EOF
        apiVersion: app.redislabs.com/v1alpha1
        kind: RedisEnterpriseDatabase
        metadata:
          name: redis-enterprise-database
        spec:
          evictionPolicy: illegal
        EOF
        ```
        
        This must fail with an error output by the admission webhook redb.admisison.redislabs that is being denied because it can't get the login credentials for the Redis Enterprise Cluster as none were specified.
        
        ```shell script
        Error from server: error when creating "STDIN": admission webhook "redb.admission.redislabs" denied the request: eviction_policy: u'illegal' is not one of [u'volatile-lru', u'volatile-ttl', u'volatile-random', u'allkeys-lru', u'allkeys-random', u'noeviction', u'volatile-lfu', u'allkeys-lfu']
        ```
       > Note: procedure to enable admission is documented with further detail [here](admission/README.md).
 
6. Redis Enterprise Database custom resource - `RedisEnterpriseDatabase`

   Create a `RedisEnterpriseDatabase` (REDB) by using Custom Resource.
   The Redis Enterprise Operator can be instructed to manage databases on the Redis Enterprise Cluster using the REDB custom resource.
    Example:
    ```yaml
    cat << EOF > /tmp/redis-enterprise-database.yml
    apiVersion: app.redislabs.com/v1alpha1
    kind: RedisEnterpriseDatabase
    metadata:
      name: redis-enterprise-database
    spec:
      memorySize: 100MB
    EOF
    kubectl apply -f /tmp/redis-enterprise-database.yml
    ```
    Replace the name of the cluster with the one used on the current namespace.
    All REDB configuration options are documented [here](redis_enterprise_database_api.md).





### Installation on OpenShift

The "OpenShift" installation deploys the operator from the current release with the RHEL image from DockerHub and default OpenShift settings.
This is the fastest way to get up and running with a new cluster on OpenShift 3.x.
For OpenShift 4.x, you may choose to use OLM deployment from within your OpenShift cluster or follow the steps below.
Other custom configurations are referenced in this repository.
If you are running on OpenShift 3.x, use the `bundle.yaml` file located under `openshift_3_x` folder (see comment in step 4).
That folder also contains the custom resource definitions compatible with OpenShift 3.x.
> Note: you will need to replace `<my-project>` with your project name.

1. Create a new project:

    ```bash
    oc new-project my-project
    ```

2. Perform the following commands (you need cluster admin permissions for your Kubernetes cluster):

    ```bash
    oc apply -f openshift/scc.yaml
    ```

    You should receive the following response:
    `securitycontextconstraints.security.openshift.io "redis-enterprise-scc" configured`

3. Provide the operator permissions for pods (substitute your project for "my-project"):

    ```bash
    oc adm policy add-scc-to-user redis-enterprise-scc system:serviceaccount:my-project:redis-enterprise-operator
    oc adm policy add-scc-to-user redis-enterprise-scc system:serviceaccount:my-project:rec
    ```
   > Note - change rec in the suffix of the 2nd command with the name of the RedisEnterpriseCluster, if different (see step "Redis Enterprise Cluster custom resource" below).

4. Deploy the OpenShift operator bundle:
    > NOTE: Update the `storageClassName` setting in `openshift.bundle.yaml` (by default its set to `gp2`).

    ```bash
    oc apply -f openshift.bundle.yaml
    ``` 
    > Note: If you are running on OpenShift 3.x, use the `bundle.yaml` file located under `openshift_3_x` folder.

5. Redis Enterprise Cluster custom resource - `RedisEnterpriseCluster`

    Apply the `RedisEnterpriseCluster` resource with RHEL7 based images:

    ```bash
    oc apply -f openshift/rec_rhel.yaml
    ```
6. Redis Enterprise Database (REDB) Admission Controller:
    The Admission Controlller is recommended for use. It uses the Redis Enterprise Cluster to dynamically validate that REDB resources as configured by the operator are valid.
    Steps to configure the Admission Controller:
    * Wait for the secret to be created by the operator bundle deployment
    ```shell script
        kubectl get secret admission-tls
        NAME            TYPE     DATA   AGE
        admission-tls   Opaque   2      2m43s
   ```
    * Enable the Kubernetes webhook using the generated certificate
      
         ```shell script
         # save cert
         CERT=`kubectl get secret admission-tls -o jsonpath='{.data.cert}'`
         sed 's/NAMESPACE_OF_SERVICE_ACCOUNT/demo/g' admission/webhook.yaml | kubectl create -f -
   
         # create patch file
         cat > modified-webhook.yaml <<EOF
         webhooks:
         - admissionReviewVersions:
           clientConfig:
             caBundle: $CERT
           name: redb.admission.redislabs
           admissionReviewVersions: ["v1beta1"]
         EOF
         # patch webhook with caBundle
         oc patch ValidatingWebhookConfiguration redb-admission --patch "$(cat modified-webhook.yaml)"
         ```
    > **Note:** If you're not using multiple namespaces you may skip to ["Verify the installation"](#verify_admission_installation_openshift) step.
   
    * Limiting the webhook to the relevant namespaces:    
      Unless limited, webhooks will intercept requests from all namespaces.<br>
      In case you have several REC objects on your K8S cluster you need to limit the webhook to the relevant namespace.  
      This is done by adding a `namespaceSelector` to the webhook spec that targets a label found on the namespace.<br>
    
         * First, make sure you have such a relevant label on the namespace and that it is unique for this namespace. e.g.
         
        ```yaml
        apiVersion: v1
        kind: Namespace
        metadata:
          labels:
            namespace-name: staging
        name: staging
        ```
        
        * Then, patch the webhook with a namespaceSelector. See this example:
        ```shell script
        cat > modified-webhook.yaml <<EOF
        webhooks:
        - name: redb.admission.redislabs
          namespaceSelector:
            matchLabels:
              namespace-name: staging
        EOF
        ```
        
        * apply the patch:
        ```shell script
        oc patch ValidatingWebhookConfiguration redb-admission --patch "$(cat modified-webhook.yaml)"
        ```
     <a name="verify_admission_installation_openshift"></a>
     * Verify the installation
        In order to verify that the all the components of the Admission Controller are installed correctly, we will try to apply an invalid resource that should force the admission controller to reject it.  If it applies succesfully, it means the admission controller has not been hooked up correctly.
        
        ```shell script
        $ oc apply -f - << EOF
        apiVersion: app.redislabs.com/v1alpha1
        kind: RedisEnterpriseDatabase
        metadata:
          name: redis-enterprise-database
        spec:
          evictionPolicy: illegal
        EOF
        ```
        
        This must fail with an error output by the admission webhook redb.admisison.redislabs that is being denied because it can't get the login credentials for the Redis Enterprise Cluster as none were specified.
        
        ```shell script
        Error from server: error when creating "STDIN": admission webhook "redb.admission.redislabs" denied the request: eviction_policy: u'illegal' is not one of [u'volatile-lru', u'volatile-ttl', u'volatile-random', u'allkeys-lru', u'allkeys-random', u'noeviction', u'volatile-lfu', u'allkeys-lfu']
        ```
      > Note: procedure to enable admission is documented with further detail [here](admission/README.md

7. Redis Enterprise Database custom resource - `RedisEnterpriseDatabase`

   Create a `RedisEnterpriseDatabase` (REDB) by using Custom Resource.
   The Redis Enterprise Operator can be instructed to manage databases on the Redis Enterprise Cluster using the REDB custom resource.
    Example:
    ```yaml
    cat << EOF > /tmp/redis-enterprise-database.yml
    apiVersion: app.redislabs.com/v1alpha1
    kind: RedisEnterpriseDatabase
    metadata:
      name: redis-enterprise-database
    spec:

      memorySize: 100MB
    EOF
    oc apply -f /tmp/redis-enterprise-database.yml
    ```
    Replace the name of the cluster with the one used on the current namespace.
    All REDB configuration options are documented [here](redis_enterprise_database_api.md).



### Installation on VMWare Tanzu
  Instruction on how to deploy the Operator on PKS can be found on the [Redis Labs documentation Website](https://docs.redislabs.com/latest/platforms/kubernetes/getting-started/tanzu/)
 
## Configuration

### RedisEnterpriseCluster custom resource
The operator deploys a `RedisEnterpriseCluster` with default configurations values, but those can be customized in the `RedisEnterpriseCluster` spec as follow:

* Redis Enterprise Image
  ```yaml
    redisEnterpriseImageSpec:
      imagePullPolicy:  IfNotPresent
      repository:       redislabs/redis
      versionTag:       6.2.10-90
  ```

* Persistence
  ```yaml
    persistentSpec:
      enabled: true
      volumeSize: "10Gi" # if you don't provide default is 5 times RAM size
      storageClassName: "standard" #on AWS common storage class is gp2
  ```

* Redis Enterprise Nodes(pods) resources
  ```yaml
    redisEnterpriseNodeResources:
      limits:
        cpu: "4000m"
        memory: 4Gi
      requests:
        cpu: "4000m"
        memory: 4Gi
  ```

* Cluster username (Default is demo@redislabs.com)
  ```yaml
  username: "admin@acme.com"
  ```

* Extra Labels: Additional labels to tag the k8s resources created during deployment
  ```yaml
    extraLabels:
      example1: "some-value"
      example2: "some-value"
  ```

* UI service type: Load Balancer or cluster IP (default)
  ```yaml
  uiServiceType: LoadBalancer
  ```

* Database service type (optional): Service types for access to databases. Should be a comma separated list. The possible values are cluster_ip, headless, and load_balancer. Default value is `cluster_ip,headless`. For example, to create a load_balancer type database service, explicitly add the following declaration to the Redis Enterprise Cluster spec:
  ```yaml
  servicesRiggerSpec:
    databaseServiceType: load_balancer
  ```

* UI annotations: Add custom annotation to the UI service
  ```yaml
    uiAnnotations:
      uiAnnotation1: 'UI-annotation1'
      uiAnnotation2: 'UI-Annotation2'
  ```

* SideCar containers: images that will run along side the redis enterprise containers
  ```yaml
    sideContainersSpec:
      - name: sidecar
        image: dockerhub_repo/repo:tag
        imagePullPolicy: IfNotPresent
  ```

* CRDB (Active Active):
  >  Currently supported for OpenShift*
  ```yaml
  activeActive: # edit values according to your cluster
    apiIngressUrl:  my-cluster1-api.myopenshiftcluster1.com
    dbIngressSuffix: -dbsuffix1.myopenshiftcluster1.com
    method: openShiftRoute
  ```


* IPV4 enforcement

  You might not have IPV6 support in your K8S cluster.
  In this case, you could enforce the use of IPV4, by adding the following attribute to the REC spec:
  ```yaml
    enforceIPv4: true
  ```
  Note: Setting 'enforceIPv4' to 'true' is a requirement for running REC on PKS.

  [requirements]: https://redislabs.com/redis-enterprise-documentation/administering/designing-production/hardware-requirements/
  [service-catalog]: https://kubernetes.io/docs/concepts/extend-kubernetes/service-catalog/

* Full detail can be found in [Redis Enterprise Cluster Custom Resource Specification](redis_enterprise_cluster_api.md).


### Private Repositories

Whenever images are not pulled from DockerHub, the following configuration must be specified:

In *RedisEnterpriseClusterSpec* (redis_enterprise_cluster.yaml):
- *redisEnterpriseImageSpec*
- *redisEnterpriseServicesRiggerImageSpec*
- *bootstrapperImageSpec*

Image specifications in *RedisEnterpriseClusterSpec* follow the same schema:
>see [ImageSpec](redis_enterprise_cluster_api.md#imagespec) for full reference

For example:
```yaml
  redisEnterpriseImageSpec:
    imagePullPolicy:  IfNotPresent
    repository:       harbor.corp.local/redisenterprise/redis
    versionTag:       6.2.10-90
```

```yaml
  redisEnterpriseServicesRiggerImageSpec:
    imagePullPolicy:  IfNotPresent
    repository:       harbor.corp.local/redisenterprise/k8s-controller
    versionTag:       6.2.10-4
```

```yaml
  bootstrapperImageSpec:
    imagePullPolicy:  IfNotPresent
    repository:       harbor.corp.local/redisenterprise/operator
    versionTag:       6.2.10-4
```

In Operator Deployment spec (operator.yaml):

For example:
```yaml
spec:
  template:
    spec:
      containers:
        - name: redis-enterprise-operator
          image: harbor.corp.local/redisenterprise/operator:6.2.10-4
```

Image specification follow the [K8s Container schema](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#container-v1-core).

### Pull secrets

Private repositories which require login can be accessed by creating a pull secret and declaring it in both the *RedisEnterpriseClusterSpec* and in the Operator Deployment spec.

[Create a pull secret](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-by-providing-credentials-on-the-command-line) by running:

```shell
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```
> NOTE: Make sure to witch context to the REC namespace or add flag -n <namespace>.

where:

-   `<your-registry-server>`  is your Private repository FQDN. ([https://index.docker.io/v1/](https://index.docker.io/v1/)  for DockerHub)
-   `<your-name>`  is your Docker username.
-   `<your-pword>`  is your Docker password.
-   `<your-email>`  is your Docker email.

This creates a pull secret names `regcred`

To use in the *RedisEnterpriseClusterSpec*:
```yaml
spec:
  pullSecrets:
    -name: regcred
```  

To use in the Operator Deployment:
```yaml
spec:
  template:
    spec:
      imagePullSecrets:
      -name: regcred
```

### Advanced Configuration
- To configure Priority Class, Node Pool, Eviction Thresholds and other advances configuration see [topics.md](topics.md) file.
- Full [Redis Enterprise Cluster Custom Resource Specification](redis_enterprise_cluster_api.md)
- Full [Redis Enterprise Database Custom Resource Specification](redis_enterprise_database_api.md)

</br> </br>

## Connect to Redis Enterprise Software web console

The username and password for the web console are stored in a secret with the Redis Enterprise Cluster name on the k8s.
in order to connect to the web console the port-forward or load balancer can be used.

First, extract the username and password from the secret: 
1. Switch to the namespace with the Redis Enterprise Cluster via the command below (replace <namespace> with the relevant namespace):
```bash
    kubectl config set-context --current --namespace=<namespace>
```
![Alt text](./images/web_console_1.png?raw=true)
2. List the secrets:
```bash
    kubectl get secret
```
![Alt text](./images/web_console_2.png?raw=true)
3. Run the `kubectl get secret` command to view the secret with the credentials (replace the <cluster name> with the name of your Redis Enterprise Cluster):
```bash
    kubectl get secret <cluster name> -o yaml
```
![Alt text](./images/web_console_3.png?raw=true)
4. Extract the username and password (replace the <cluster name> with the name of your Redis Enterprise Cluster):

```bash
    kubectl get secret <cluster name> -o jsonpath='{.data.username}' | base64 --decode
    kubectl get secret <cluster name> -o jsonpath='{.data.password}' | base64 --decode
```
![Alt text](./images/web_console_4.png?raw=true)

Connect to the web console with one of the two following methods:

Method 1: using port-forward
1. Get the port of the cluster UI service via the command below (replace the <cluster name> with the name of your Redis Enterprise Cluster):
```bash
    kubectl get service/<cluster name>-ui -o yaml
```
Note: the default port is 8443.
![Alt text](./images/web_console_5.png?raw=true)
2. Run the kubectl port-forward service to set port-forward. Replace the <cluster name> with the name of your Redis Enterprise Cluster, replace <service port> with the port of the service, and replace <local port> with the port you want to use on the local machine.
```bash
    kubectl port-forward service/<cluster name>-ui <local port>:<service port>
```
![Alt text](./images/web_console_6.png?raw=true)
3. In the web browser on the local machine to see the Redis Enterprise web console go to:
https://localhost:<local port>
Don't forget to replace the <local port> with the one used in the previous command.
![Alt text](./images/web_console_7.png?raw=true)

Method 2: load balancer
> <note> Configuring a load balancer service for the UI will create an external IP address, widely available (when set on cloud providers which support external load balancers). Use with caution. </note>
1. Run the command below to set the UI service type as load balancer, replace the <cluster name> with the name of your Redis Enterprise Cluster:
```bash
    kubectl patch rec <cluster name> --type merge --patch "{\"spec\":{\"uiServiceType\":\"LoadBalancer\"}}"
```
![Alt text](./images/web_console_8.png?raw=true)
2. Get the external IP and service port of the service via the command below:
```bash
    kubectl get service/<cluster name>-ui
```
Note: the default port is 8443.
![Alt text](./images/web_console_9.png?raw=true)
3. View the web console from the web browser on your local machine:
https://<external IP>:<service port>
Don't forget to replace the <external IP> and <service port> with the values from the previous step.
![Alt text](./images/web_console_10.png?raw=true)

Note: in the  examples above the Redis Enterprise Cluster name is: 'rec' and the namespace is 'demo'.

</br> </br>

## Upgrade

The Operator automates and simplifies the upgrade process.  
The Redis Enterprise Cluster Software, and the Redis Enterprise Operator for Kubernetes versions are tightly coupled and should be upgraded together.  
It is recommended to use the bundle.yaml to upgrade, as it loads all the relevant CRD documents for this version. If the updated CRDs are not loaded, the operator might fail.
There are two ways to upgrade - either set 'autoUpgradeRedisEnterprise' within the Redis Enterprise Cluster Spec to instruct the operator to automatically upgrade to the compatible version, or specify the correct Redis Enterprise image manually using the versionTag attribute. The Redis Enterprise Version compatible with this release is 6.2.10-90

```yaml
  autoUpgradeRedisEnterprise: true
```

Alternatively:
```yaml
  RedisEnterpriseImageSpec:
    versionTag: redislabs/redis:6.2.10-90
```

## Supported K8S Distributions
Each release of the Redis Enterprise Operator deployment is thoroughly tested against a set of Kubernetes distributions. The table below lists these, along with the current release's support status. "Supported", as well as "deprecated" support status indicates the current release has been tested in this environment and supported by RedisLabs. "Deprecated" also indicates that support will be dropped in a coming future release. "No longer supported" indicates that support has been dropped for this distribution. Any distribution that isn't explicitly listed is not supported for production workloads by RedisLabs.
Supported versions (platforms/versions that are not listed are not supported): 
| Distribution                    | Support Status |
|---------------------------------|----------------|
| Openshift 3.11 (K8s 1.11)       | deprecated     |
| OpenShift 4.6  (K8s 1.19)       | supported      |
| OpenShift 4.7  (K8s 1.20)       | supported      |
| OpenShift 4.8  (K8s 1.21)       | supported      |
| OpenShift 4.9  (K8s 1.22)       | supported      |
| KOPS vanilla 1.18               | supported      |
| KOPS vanilla 1.19               | supported      |
| KOPS vanilla 1.20               | supported      |
| KOPS vanilla 1.21               | supported      |
| KOPS vanilla 1.22               | supported      |
| GKE 1.19                        | supported      |
| GKE 1.20                        | supported      |
| GKE 1.21                        | supported      |
| GKE 1.22                        | supported      |
| Rancher 2.5 (K8s 1.17)          | *deprecated    |
| Rancher 2.5 (K8s 1.18)          | *deprecated    |
| Rancher 2.5 (K8s 1.19)          | *deprecated    |
| Rancher 2.5 (K8s 1.20)          | *deprecated    |
| Rancher 2.6 (K8s 1.19)          | supported      |
| Rancher 2.6 (K8s 1.20)          | supported      |
| Rancher 2.6 (K8s 1.21)          | supported      |
| VMWare TKGIE** 1.10 (K8s 1.19)  | supported      |
| VMWare TKGIE 1.11 (K8s 1.20).   | supported |
| AKS 1.19                        | deprecated     |
| AKS 1.20                        | supported      |
| AKS 1.21                        | supported      |
| AKS 1.22                        | supported      |
| EKS 1.18                        | supported      |
| EKS 1.19                        | supported      |
| EKS 1.20                        | supported      |
| EKS 1.21                        | supported      |

\* No longer supported by the vendor
\*\* Tanzu Kubernetes Grid Integrated Edition
