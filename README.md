# SolidFire Operator

**WARNING: This project is experimental - do not deploy to production clusters**

- [SolidFire Operator](#solidfire-operator)
  - [What is this](#what-is-this)
  - [Install Solidfire Operator](#install-solidfire-operator)
  - [Use it](#use-it)
  - [Lose it](#lose-it)
  - [Develop](#develop)
    - [Use another language or operator framework](#use-another-language-or-operator-framework)
  - [FAQs](#faqs)
  - [Acknowledgements](#acknowledgements)
  - [Example](#example)

## What is this

This repository and related Docker Hub images aim to provide a working example for building and using a SolidFire Operator for Kubernetes.

The first release covers basic operations on [the SolidFire QoS Policy object](https://docs.netapp.com/us-en/element-software/api/reference_element_api_qos.html) and can be easily expanded.

## Install Solidfire Operator

On a UNIXy system with access to your Kubernetes cluster, pick a SolidFire Operator version (e.g. 0.0.4), clone this repository and deploy.

```sh
git clone https://github.com/scaleoutsean/solidfire-operator
cd solidfire-operator
make deploy VERSION=$VERSION
```

Operator container will be downloaded from [https://hub.docker.com/r/scaleoutsean/solidfire-operator](https://hub.docker.com/r/scaleoutsean/solidfire-operator), but you can build your own and use it instead.

If deployment has been successful, we should be able to see a deployment and pods.

```
$ kubectl get deployments -n solidfire-operator-system
NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
solidfire-operator-controller-manager   1/1     1            1           8m32s

$ kubectl get pods -n solidfire-operator-system
NAME                                                    READY   STATUS    RESTARTS   AGE
solidfire-operator-controller-manager-85964766c-ncll2   2/2     Running   0          8m1s
```

## Use it

**WARNING:** CRD-related crap can be difficult to remove, especially if `kubectl delete` gets stuck, blocking removal of the operator, so think twice before you try this on a production cluster.

Edit the YAML sample file from `config/samples` and give it a try.

```dotnetcli
vim config/samples/solidfire_v1alpha1_qospolicy.yaml
kubectl apply -f solidfire-operator/config/samples/solidfire_v1alpha1_qospolicy.yaml
```

In order to avoid getting stuck, make sure `kubectl delete -f` completes successfully. It usually takes 10-20 seconds because reconciliation happens periodically.

## Lose it

From the cloned repository folder, run `make` with undeploy and uninstall commands:

```
make uninstall
make undeploy
```

If any of these hangs, maybe you have some pending operation that's stuck ("Warning: Detected changes to resource something-something which is currently being deleted."). This may happen if you interrupt a delete resource command, for example. Usually re-runing the commands twice helps.

## Develop

If you don't have a SolidFire cluster, get the SoliFire Demo VM from NetApp Support site (Tools section; registration required).

Clone the repo, and RTFM to see what's what:

- [Operator Framework](https://operatorframework.io)
- [Ansible Collection for SolidFire](https://galaxy.ansible.com/netapp/elementsw)

You need to check the Ansible-related documentation of Operator Framework and get the right kubectl for your environment. I used it v1.23.6, but that's not so important for SolidFire - this operator only depends on Python, Ansible and SolidFire Collection for Ansible, all of which are defined in Dockerfile. kubectl can be any supported and this operator framework supports Kubernetes on ARM64 (Trident CSI v22.01 doesn't support ARM64, but [can be made to run](/2021/02/24/netapp-trident-on-arm64.html) on it).

### Use another language or operator framework

If you know Go I would suggest using own code (see [Terraform Provider for SolidFire](https://github.com/NetApp/terraform-provider-netapp-elementsw/) for examples of using Go with "raw" SolidFire API). .NET developers may want to check [other approaches](https://github.com/buehler/dotnet-operator-sdk) where [SolidFire .NET SDK](https://solidfire.github.io/sdk-dotnet/) could be used to eliminate writing boilerplate code.

In other words, you don't have to use Ansible and you don't have to use this Operator Framework either - there's half a dozen of them out there - to build your own SolidFire operator.

## FAQs

**Q:** What does creating a QoS policy on a SolidFire cluster used by Kubernetes do?

A: Nothing, and that's the point. NetApp Trident CSI is not aware of of changes made to PV settings by bypassing Trident (and in the case of QoS Policy that's what would happen). So this initial example does nothing on purpose.

**Q:** If I can't use a SolidFire Operator to make any changes, what's the point?

A: The point is there are areas of SolidFire configuration that Trident is not aware of. Those are safe and even desirable to operate on, as long as we know what we're doing.

**Q:** Ideas?

A: Set up SolidFire replication and perform SolidFire storage cluster failover.

**Q:** What if I don't use Trident CSI?

A: You can use an operator even if you use Host Path volumes. Or if you use Cinder CSI with SolidFire. Or if you have the strange desire to manage SolidFire from Kubernetes while not using Kubernetes for your workloads, which paradoxically makes sense because your Master Node(s) with etcd on local disks would ensure your management application runs even when storage fails over or goes down.

## Acknowledgements

Thanks to @vrd83 for the inspiration to give this idea a try.

## Example

This is an example of an edited config/samples/solidfire_v1alpha1_qospolicy.yaml with the password string replaced by asterisks. `status: present` is default so there's no need to have it in there in order to create a SolidFire QoS Policy. 

(The FQDN in apiVersion value is arbitrary; solidfire-operator doesn't "ping" the FQDN or send any info. Check Operator Framework's Privacy Policy to see if they do any data collection.)

```yaml
apiVersion: solidfire.datafabric.club/v1alpha1
kind: QosPolicy
metadata:
  name: qospolicy-sample
spec:
  hostname: "192.168.1.30"
  username: "admin"
  password: "*****"
  name: "sample"
  qos:
    min: 111
    max: 222
    burst: 333
```

With the edited QoS Policy sample file:

```sh
$ kubectl create -f config/samples/solidfire_v1alpha1_qospolicy.yaml 
qospolicy.solidfire.datafabric.club/qospolicy-sample created

$ kubectl get qospolicy.solidfire.datafabric.club
NAME               AGE
qospolicy-sample   1s

$ kubectl describe qospolicy.solidfire.datafabric.club qospolicy-sample
Name:         qospolicy-sample
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  solidfire.datafabric.club/v1alpha1
Kind:         QosPolicy
Metadata:
  Creation Timestamp:  2022-04-27T10:57:43Z
  Finalizers:
    solidfire.datafabric.club/finalizer
  Generation:  1
  Managed Fields:
    API Version:  solidfire.datafabric.club/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:finalizers:
          .:
          v:"solidfire.datafabric.club/finalizer":
    Manager:      ansible-operator
    Operation:    Update
    Time:         2022-04-27T10:57:43Z
    API Version:  solidfire.datafabric.club/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:conditions:
    Manager:      ansible-operator
    Operation:    Update
    Subresource:  status
    Time:         2022-04-27T10:57:43Z
    API Version:  solidfire.datafabric.club/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        .:
        f:hostname:
        f:name:
        f:password:
        f:qos:
          .:
          f:burst:
          f:max:
          f:min:
        f:username:
    Manager:         kubectl-create
    Operation:       Update
    Time:            2022-04-27T10:57:43Z
  Resource Version:  1719310
  UID:               478440b9-87cf-4e87-b707-94e24b3da803
Spec:
  Hostname:  192.168.1.30
  Name:      sample
  Password:  admin
  Qos:
    Burst:   333
    Max:     222
    Min:     111
  Username:  admin
Status:
  Conditions:
    Last Transition Time:  2022-04-27T10:57:48Z
    Message:               
    Reason:                
    Status:                False
    Type:                  Failure
    Ansible Result:
      Changed:             0
      Completion:          2022-04-28T02:49:11.7511
      Failures:            0
      Ok:                  2
      Skipped:             0
    Last Transition Time:  2022-04-27T10:57:43Z
    Message:               Awaiting next reconciliation
    Reason:                Successful
    Status:                True
    Type:                  Running
    Last Transition Time:  2022-04-28T02:49:11Z
    Message:               Last reconciliation succeeded
    Reason:                Successful
    Status:                True
    Type:                  Successful
Events:                    <none>

$ kubectl delete -f config/samples/solidfire_v1alpha1_qospolicy.yaml 
```