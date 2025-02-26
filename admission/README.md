# REDB Admission Controller

Redis Labs' Redis Enterprise Operator provides an installable admission control that can be used to verify RedisEnterpriseDatabase resources on creation and modification for correctness.  This prevents end users from creating syntatically valid but functionally invalid database configurations.  The admission control leverages Kubernetes' built in [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/).

**Note:** Redis Labs' Redis Enterprise Operator can also be installed through the [Gesher Admission Proxy](GESHER.md) 

## Hooking up the Admission controller directly with Kubernetes
**NOTE**: This only has to be done the first time setting up the redis enterprise operator, it can be skipped on update

1. Wait for the secret to be created

    ```shell script
    kubectl get secret admission-tls
    NAME            TYPE     DATA   AGE
    admission-tls   Opaque   2      2m43s
    ```

2. Enable the Kubernetes webhook using the generated certificate stored in a kubernetes secret

      **NOTE**: One must replace REPLACE_WITH_NAMESPACE in the following command with the namespace the REC was installed into.

      ```shell script
      # save cert
      CERT=`kubectl get secret admission-tls -o jsonpath='{.data.cert}'`
      sed 's/NAMESPACE_OF_SERVICE_ACCOUNT/REPLACE_WITH_NAMESPACE/g' webhook.yaml | kubectl create -f -

      # create patch file
      cat > modified-webhook.yaml <<EOF
      webhooks:
      - name: redb.admission.redislabs
        clientConfig:
          caBundle: $CERT
        admissionReviewVersions: ["v1beta1"]
      EOF
      # patch webhook with caBundle
      kubectl patch ValidatingWebhookConfiguration redb-admission --patch "$(cat modified-webhook.yaml)"
      ```
          
## Verifying Installation

In order to verify that the all the components of the Admission Controller are installed correctly, we will try to apply an invalid resource that should force the admission controller to reject it.  If it applies succesfully, it means the admission controller has not been hooked up correctly.

```shell script
$ kubectl apply -f - << EOF
apiVersion: app.redislabs.com/v1alpha1
kind: RedisEnterpriseDatabase
metadata:
  name: redis-enterprise-database
spec:
  evictionPolicy: illegal
  defaultUser: false
EOF
```

This must fail with an error output by the admission webhook redb.admisison.redislabs that is being denied because it can't get the login credentials for the Redis Enterprise Cluster as none were specified.

```shell script
Error from server: error when creating "STDIN": admission webhook "redb.admission.redislabs" denied the request: eviction_policy: u'illegal' is not one of [u'volatile-lru', u'volatile-ttl', u'volatile-random', u'allkeys-lru', u'allkeys-random', u'noeviction', u'volatile-lfu', u'allkeys-lfu']
```
