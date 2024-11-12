---
date: 2023-03-31
title: Using Custom Resources (CRs) as Persistence Layer for applications in Kubernetes
description: Data persistence on Kubernetes for cloud native applications using Kubernetes CRs
image: images/blog/use-crds-as-persistence-layer-for-applications/undraw_advanced_customization.svg

cover_image: false
cover_image_src: 
cover_image_height: ""
cover_image_width: ""

author: mahendra-bagul
series: Cloud Native Applications
categories:
- Cloud Native
- Kubernetes
- Low-Code
- Applications

tags:
- Application Development
- Low-Code
- Kubernetes Applications Development
- Cloud Native Applications

# image color code in undraw.co #FB7E44 
feedback: false
draft: false

---

{{< image src="images/blog/use-crds-as-persistence-layer-for-applications/undraw_advanced_customization.svg" alt="alter-text" height="" width="200px" class="img-fluid" caption="" webp="false" position="float-left" >}}

We have been working on an open source project which runs on K8s cluster. We had a need to persist some data, we had two
options. Either use a full-fledged database or store the data in some files in plain text format.

We came to know that the data can be persisted on K8s cluster in the form of K8s custom resources. As our solution runs
on K8s so, we thought to leverage K8s custom resources as a persistence layer.

If you are new to K8s custom-resources, please read on below link.

- https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions.

The main difference between CR(custom resource) and CRD(custom resource definition) is that the CRD is a schema whereas
CR represents a resource matching that CRD.

### Setup KinD based K8s cluster

For this blog, let's use KinD to create a local K8s cluster. Please follow steps given on below links.

- Install KinD from https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries
- Create KinD cluster https://kind.sigs.k8s.io/docs/user/quick-start/#creating-a-cluster
- Check if you can access the cluster created in previous step, and you are able to list down the pods.

### Code walkthrough

All the code is present [here](https://github.com/intelops/k8s-custom-resource-demo). The project consists of ExpressJS
with typescript as a choice of language. The structure of the repository looks like below,

- crds - containing the CRD (yaml specification) for the custom resource for which we have a REST CRUD operations
  implemented.
- src/store/k8s-client - containing the client code connecting to k8s and performing CRUD operations.
- other standard folders required for REST services implementation in ExpressJS.

The employee-crd.yaml looks like below.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: employees.capten.ai
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: capten.ai
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: employees
    # singular name to be used as an alias on the CLI and for display
    singular: employee
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: Employee
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
      - emp
  # either Namespaced or Cluster
  scope: Namespaced
  versions:
    - name: v1alpha1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                role:
                  type: string
          required: [ "spec" ]
```

Before we run the code, we have to apply the CRD onto the cluster like this.

```shell
kubectl apply -f crds/employee-crd.yaml
```

We can now create `employee` custom resources in the same fashion we do for in-built resources. In other words, all the
kubectl
commands which work with in-built resources, the same would work with your custom resources too.

```shell
kubectl get employee -A
```

### Node client to K8s - @kubernetes/client-node

To perform CRUD operations on the custom resources programmatically, we have
used [`@kubernetes/client-node`](https://www.npmjs.com/package/@kubernetes/client-node) npm package. We have to
initialise K8s client in the package and it's done by adding below snippet in the
code.

```typescript
import * as k8s from "@kubernetes/client-node";

const kubeConfig = new k8s.KubeConfig();
if (process.env.NODE_ENV === 'development') {
    kubeConfig.loadFromDefault();
} else {
    kubeConfig.loadFromCluster();
}
const client = kubeConfig.makeApiClient(k8s.CustomObjectsApi);
```

Once the client is created using KubeConfig, we can use the methods available to perform actions on K8s resources(
including custom resources).

### Methods to perform CRUD operations on K8s custom resource

#### To create a custom resource

```typescript
await client.createNamespacedCustomObject(group, version, namespace, plural, JSON.parse(payload));
```

#### To retrieve a custom resource

```typescript
await client.getNamespacedCustomObject(group, version, namespace, plural, name);
```

#### To update the given custom resource

Here the patch operation needs special treatment, the options needs a special header here like below

```typescript
const options = {"headers": {"Content-type": k8s.PatchUtils.PATCH_FORMAT_JSON_PATCH}};
```

and the patch contains below

```typescript
const patch = [{
    "op": "replace",
    "path": "/spec",
    "value": JSON.parse(payload)
}];
```

The path contains the key which is getting replaced/patched.

```typescript
await client.patchNamespacedCustomObject(group, version, namespace, plural, name, JSON.parse(patch), undefined, undefined, undefined, options);
```

#### To list down all the custom resources

```typescript
await client.listNamespacedCustomObject(group, version, namespace, plural, "true", false, "", "", labelSelector);
```

#### To delete the given custom resource

```typescript
await client.deleteNamespacedCustomObject(group, version, namespace, plural, name)
```

With above client code added, it can now be invoked from http handlers. You can check them here.

Run the code cloned by following instructions given [`here`](https://github.com/intelops/k8s-custom-resource-demo#run-code-on-local). You can now use curl commands to create/update/read/delete custom resources.

### Create an employee

- Fire below command to create an employee.

```shell
curl -X POST -H "Content-Type: application/json" -d '{"name":"Mahendra","role": "developer"}' http://localhost:8080/v1/employees
```

- You can check if the employee was created by firing below command.

```shell
kubectl get employee -A
```

Similarly, you can call other rest apis too which will retrieve the data from K8s.

### Conclusion

We saw that how we can use K8s custom resources to persist the application data when deploying a full-fledged database
is not an option. Let us know your thoughts on this.
