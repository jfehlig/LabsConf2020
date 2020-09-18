# Introduction

The rapid adoption of Kubernetes in the IT landscape has spurned the need to run traditional virtualized workloads on Kubernetes clusters, using the same tools for managing container-native workloads. There are a few projects projects providing this feature, with KubeVirt gaining the most traction.

This paper will describe the journey of a few old-school virtualization hackers as they make their way into the world of containerization, Kubernetes, and running virtual machines in containers. The journey will start with a quick survey of open source projects providing virtualization support on Kubernetes, then venture into KubeVirt and the process of painting it green. The journey continues by exploring the installation of KubeVirt on a Kubernetes cluster and concludes with an excursion into creating and interacting with virtual machines.

Before beginning it will be useful to clarify a few terms used throughout the journey. The term 'virtual machine' will be used to refer to a collection of virtual resources that together present a machine in which a guest OS can be installed. Likely most readers are familiar with this notion of a virtual machine from using KVM, Xen, or VMware. KubeVirt also uses the term VirtualMachine (VM) to refer to a custom Kubernetes resource the encapsulates a virtual machine. KubeVirt also provides a VirtualMachineInstance custom resource that encapsulates a running VM. The Kubernetes term 'pod' is also used throughout the journey. A pod is the smallest unit of work that Kubernetes will schedule. Pods typically consist of multiple containers that generally share a network namespace but have their own PID and mount namespaces as well as CGroups for CPU, memory and devices.


# Opensource Projects Extending Kubernets with Virtualization Support

The journey started during Hackweek 19 by quickly looking at two minimally functional projects providing container and virtual machine convergence. Virtlet is a Mirantis sponsored project that essentially transforms a Kubernetes node from one that run containers to one that runs virtual machines. Virtlet enables virtualization in Kubernetes by replacing the container runtime with one tailored for virtualization. With this approach any Kubernetes node running Virtlet in effect restricted to running virtual machine workloads. Mirantis also provides a CRI proxy that can forward container runtime RPCs to Virtlet or dockershim, but this approach excludes support for other CRI-compliant runtimes. The omission of cri-o is particularly interesting to SUSE since it's the default container runtime in CaaSP.

![alt text](Images/virtlet.png)

Replacing the container runtime, which is at the foundation of the Kubernetes architecture, seemed a dubious proposition at best. Considering Virtlet is also quite immature with limited functionality and a stagnant upstream community (last commit in December 2019, last release in May 2019), its exploration was short-lived.

The journey's next stop was at KubeVirt. KubeVirt is a Red Hat sponsored project that extends Kubernetes by adding additional virtualization resource types through Kubernetes' Custom Resource Definitions (CRD) API. By using this mechanism, the Kubernetes API can be used to manage VM resources alongside other resources Kubernetes provides. Along with the CRDs, KubeVirt provides function and business logic for the Kubernetes cluster by running additional controllers and agents

![alt text](Images/kubevirt.png)

Although the wary travelers had little experience with Kubernetes, KubeVirt seemed to fit more naturally with the Kubernetes, using the mechanisms it provides to extend the cluster functionality. From here the journey explores KubeVirt in more depth and even establishes some new routes by painting KubeVirt green.


A closer look at KubeVirt

KubeVirt extends Kubernetes by adding a custom resource definition and related logic extensions to the cluster. The custom resource definition essentially registers a new endpoint with the Kubernetes API server. Requests against this endpoint are handled by virt-api and ultimately virt-controller. The KubeVirt CRD is rather simple, adding the new endpoint KubeVirt.io

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  labels:
    operator.kubevirt.io: ""
  name: kubevirts.kubevirt.io
spec:
  additionalPrinterColumns:
  - JSONPath: .metadata.creationTimestamp
    name: Age
    type: date
  - JSONPath: .status.phase
    name: Phase
    type: string
  group: kubevirt.io
  names:
    categories:
    - all
    kind: KubeVirt
    plural: kubevirts
    shortNames:
    - kv
    - kvs
    singular: kubevirt
  scope: Namespaced
  version: v1alpha3
  versions:
  - name: v1alpha3
    served: true
    storage: true
```

Once the CRD is registered with Kubernetes, an instance can be created in the KubeVirt namespace with the following manifest

```yaml
apiVersion: KubeVirt.io/v1alpha3
kind: KubeVirt
metadata:
  name: KubeVirt
  namespace: KubeVirt
spec:
  certificateRotateStrategy: {}
  configuration: {}
  imagePullPolicy: IfNotPresent
```

Custom resources can now be CRUD using the endpoint extension in the resource's apiVersion setting. The following example creates a VirtualMachineInstance custom resource against the KubeVirt.io entrypoint

```yaml
apiVersion: KubeVirt.io/v1alpha3
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-host-disk
  name: sles15sp2
spec:
...
```

Attempting to create a custom resource that is not supported by the KubeVirt.io endpoint results in an error

```
# cat crap.yaml
apiVersion: KubeVirt.io/v1alpha3
kind: VirtualBoxInstance
...
# k apply -f test.yaml 
error: unable to recognize "test.yaml": no matches for kind "VirtualBoxInstance" in version "KubeVirt.io/v1alpha3"
```

The custom resources supported by KubeVirt are provided by the logic components: virt-api, virt-controller, virt-handler, and virt-launcher. In addition, virt-operator implements a Kubernetes Deployment resource that has the task of managing the lifecycle of the logic components.  Once deployed, the operator has sufficient knowledge to deploy the other components, extending the cluster's capabilities with virtualization support.

virt-controller and virt-api are cluster components that handle cluster-wide requests against the KubeVirt custom resources. virt-handler and virt-launcher are node components providing logic to launch, configure, monitor, and interact with virtual machines. Adding new custom resources or modifying existing ones requires changing the cluster-wide components and likely the node components as well.

The next point of interest in the journey is the KubeVirt project itself. KubeVirt is hosted on github and follows a merge request workflow. Pull requests must be '/approved' by maintainers and all gate tests must pass before the request is merged. Experience has shown some tests are quite flaky and often need a '/retest' or two before they pass.

Since KubeVirt is sponsored by Red Hat, it is no surprise that a majority of the upstream community consists of Red Hat employees. Nonetheless the community is open and welcoming to newcomers and the primary contributors have been courteous and helpful. The community uses the #virtualization channel on Slack for chat and a Google groups list for more traditional email communications.

Like many projects in the Kubernetes ecosystem, KubeVirt is written in go. It uses bazel to build all code and containers. Upstream builds using the standard make target need network access to download build containers and various bazel and go dependencies. Nearly all build targets use a pre-built container called "builder" that provides the basic build environment.

The builder container can be created from the top level Makefile with the 'builder-build' target, e.g. 'make builder-build'. Likewise the builder container can be published to Docker Hub with the 'make builder-publish' target. When building kubevirt with the builder container, a persistent docker volume is created for rsync'ing the KubeVirt sources to the container and extracting the artifacts after the build. The default make target uses the builder container to build all the go components and container images for running them. Along with the containers, the build also produces manifests for deploying KubeVirt on a Kubernetes cluster.


# Painting KubeVirt green

With SUSE's move away from OpenStack and commitment to Kubernetes and containerization came the need to support virtualized workloads on Kubernetes. Product management recognized the need and created ECOs 2411 and 2415 to provide KubeVirt on SLE15 SP2. The virtualization team was tasked to implement the ECOs. The path became very steep in this part of the journey. At times the travelers were on hands and knees, crawling at a snail's pace.

The first steps on this path were used to gain a brief understanding of the build system, which was previously described. Currently only Fedora-based builder containers exist on Docker Hub. Some effort was spent creating an openSUSE Leap 15.2 based builder image, which was then successfully used to build KubeVirt, but there has been no subsequent effort to merge the work upstream. It would be considered a lower priority task since there seems little to gain by the effort and upstream acceptance is questionable.

The downstream part of the path was certainly not downhill. It was this section of the journey that required the most crawling. The first obstacle was enabling off-line building required by the build service. Bazel supports an off-line build mode that uses a previously assembled tarball of all dependencies. The tarball quickly grew, nearing a whopping 10G, before the travelers decided to pursue another route. A closer look at the hack/build-go.sh helper script revealed the KubeVirt binaries could be build directly with

    ./hack/build-go.sh install cmd/$KubeVirt-componenet && ./hack/build-copy-artifacts.sh

This route turned out to be much more promising as it avoided the bazel obstacle altogether, along with the need for a huge tarball of build dependencies. The build-go.sh helper uses the familiar `go_build` command to build the components, which are then copied to a standard location in the build tree for installation. With this discovery in hand, building the KubeVirt container images in the build service became a much easier task.

In order to begin satisfying ECOs 2411 and 2415, packages were created in IBS for virt-api, virt-controller, virt-handler, virt-launcher, and virt-operator. Initially a release tarball of the code was copied to each package and a multi-stage docker build was performed. In the first stage, the related binary was built using build-go.sh and build-copy-artifacts.sh. The container image was built in the second stage, which included installing the binary from the first stage. The approach worked well but duplicate copies of the tarball across several packages was burdensome. To optimize, a kubevirt package was created where a single copy of the KubeVirt tarball is used to build intermediate packages used solely for the purpose of installation during container build. A source service was created to automatically download new release tarballs and annotate the changelog with release highlights. At this point the tired travelers enjoyed a small moment of triumph! All KubeVirt components were being built against SLE15 SP2 and being installed when assembling SLE15 SP2 based containers. Alas a green KubeVirt was available in registry.suse.de for use by other adventurous travelers. For the virtualization team there were still obstacles to overcome.


# Deploying KubeVirt

The journey continued with exploring mechanisms to deploy KubeVirt. One obvious requirement is a working Kubernetes cluster. The SUSE CaaSP product provides a Kubernetes platform and was used to provide a Kubernetes cluster for the exploration of KubeVirt. Adventurers wanting to explore on their own can refer to a howto on installing CaaSP and KubeVirt on a collection of SLES15 SP2 hosts[2].

As already mentioned, the KubeVirt operator component implements a Kubernetes Deployment resource [4] that manages the lifecycle of the KubeVirt functional components. Deploying the operator will subsequently deploy KubeVirt, adding virtualization capabilities to the cluster. The operator is deployed to the cluster by applying a manifest that describes a desired cluster state and how to reach that state. The manifest includes the location and SHAs of the functional KubeVirt components, allowing the operator to retrieve and deploy the components to achieve the desired cluster state.

When the travelers discovered the manifest must be modified to reference the SLE-based containers, they took the easy path and manually adjusted the manifest. A future excursion will investigate mechanisms for substituting the registry and container SHAs in the manifest as they become known following container build and publish. Another adventure that lies ahead is creating a helm chart for KubeVirt. Kubernetes services are typically managed with a "Kubernetes package manager" and helm is a widely used one that has also been adopted by SUSE.

With an updated operator manifest in hand, the operator was deployed to a CaaSP4.5 cluster running on SLES15 SP2. The travelers could set back, have a drink, and watch the magic occur as it deployed KubeVirt to the cluster. Their palettes were barely wet when they noticed the virt-operator container in a failed state. Much time was wasted digging through logs only to find the container runtime on the cluster nodes failed to verify the certificate from registry.suse.de, which prevented pulling the operator image from the registry. Again the easy route was taken and registry.suse.de was added to the 'registries' and 'insecure_registries' lists in /etc/crio/crio.conf.d/00-default.conf on each cluster node. The operator could now be pulled from the registry and work its magic. Soon the travelers noticed virt-api and virt-controller containers appearing in the cluster!

Yet again the excitement was short lived, with the virt-controller container entering a failed state. This obstacle proved quite difficult to overcome and the travelers seeked aid from the CaaSP team. Not surprisingly, some KubeVirt components need special privileges to construct a container for running a virtual machine. The virtual machine container needs access to a container providing its network endpoints, potentially a container providing storage, etc. Also not surprising, the default cluster policy does not give containers such privileges. The CaaSP team helped modify the cluster security policy to give virt-operator, virt-controller, and virt-handler the necessary privileges. Typical for the journey, the easy route was taken and the CaaSP privileged pod security policy was used. A future adventure will explore creating a proper Pod Security Policy (PSP) for KubeVirt. The PSP could be included as part of the operator manifest and applied to the cluster when deploying the operator.

Alas, with the proper security policy in place on the cluster, the operator was able to successfully deploy KubeVirt! It was time for the travelers to take the final few steps to reach their goal of running virtual machines on Kubernetes!


# Running Virtual Machines

Two of the most interesting custom resources provided by KubeVirt are VirtualMachine (VM) and VirtualMachineInstance (VMI). As the name implies, a VMI is a running instance of a VM. The lifeclycle of a VMI can be managed independently from a VM, but long-lived, stateful virtual machines are managed as a VM. The VM is deployed to the cluster in a shutoff state, then activated by changing the desired state to running. Changing a VM resource state can be done with the standard Kubernetes client tool `kubectl` or with the KubeVirt provided client `virtctl`.

The VM and VMI custom resources make up part of the KubeVirt API [1]. To create a virtual machine, a VM or VMI manifest must be created that adheres to the API. The API supports setting a wide variety of the common virtual machine attributes, e.g. model of vCPU, number of vCPUs, amount of memory, disks, network ports, etc. Below is a simple example of a VMI manifest for a virtual machine with one Nehalem CPU, 2G of memory, one disk, and one network interface

```yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-host-disk
  name: sles15sp2
spec:
  domain:
    cpu:
      model: Nehalem-IBRS
    devices:
      disks:
      - disk:
          bus: virtio
        name: host-disk
      interfaces:
        - name: green
          masquerade: {}
          ports:
            - port: 80
    machine:
      type: ""
    resources:
      requests:
        memory: 2048M
  terminationGracePeriodSeconds: 0
  networks:
  - name: green
    pod: {}
  volumes:
  - hostDisk:
      path: /hostDisks/sles15sp2/disk.raw
      type: Disk
      shared: true
    name: host-disk
status: {}
```

Applying this VMI manifest to the cluster will create a virt-launcher container running libvirt and qemu, providing the familiar KVM virtual machine environment we all know and love

```
kubectl apply -f sles15sp2vmi.yaml
kubectl get vmis
kubectl get pods
```

Similar to other Kubernetes resources, VMs and VMIs can be managed with the kubectl client tool. Any kubectl operation that works with resource types will work with the KubeVirt custom resources, e.g. describe, delete, get, log, patch, etc. VM resources are a bit more awkward to manage with kubectl. Since a VM resource can be in a shutoff state, turning it on requires patching the manifest to change the desired state to running. E.g.

    kubectl patch vm sles15sp2 --type merge -p '{"spec":{"running":true}}'

virtctl is a client tool provided by the KubeVirt project that adds syntactic sugar on top of kubectl for VM and VMI resources, allowing them to be stopped, started, paused, unpaused, and migrated. virtctl also provides access to the virtual machine's serial console and graphics server.

```
virtctl start VM
virtctl console VMI
virtctl stop VM
virtctl pause VM|VMI
```

Disk and storage options for KubeVirt are quite vast and deserve their own presentation. Nonetheless the options will be briefly mentioned here. KubeVirt supports the notion of volumes and disks. Volume resources must be backed by a data source, which is typically provided by the various storage services available for Kubernetes. Supported data sources for volume resources include cloudInitNoCloud, cloudInitConfigDrive, PersistentVolumeClaim (PVC), dataVolume, ephemeral, containerDisk, hostDisk, and a few others that are less interesting.

Volume resources are mapped to disks in a VM or VMI manifest, which provide storage devices for virtual machines. The disk types supported are LUN (exposes the volume as a LUN to the VMI), disk (exposes the volume as an ordinary disk device to the VMI), and CDROM (you get the picture). The explorers experimented with hostDisk and PVC. hostDisk is quite easy to use. Simply copy an existing virtual machine image to the same path on each node in the cluster and reference the path when specifying the hostDisk volume in the VM or VMI manifest

```yaml
  volumes:
  - hostDisk:
      path: /hostDisks/sles15sp2/disk.raw
```

PVCs are likely the most useful data source for KubeVirt volume resources, particularly for traditional, stateful virtual machines. As the name suggests, PersistenVolumeClaims persist across VM restarts. Also as the name suggests, it is just a claim, a request for storage by the user. The actual storage is provided by a PersistentVolume (PV), which is a chunk of storage in the cluster that has been provisioned manually by an administrator, or dynamically using a Storage Class. An analogy can be drawn between PVCs and pods: pods consume node resources and PVCs consume PV resources.

There are a variety of storage options available for use with KubeVirt. The explorers have much to investigate [3] on this topic and encourage the interested reader to do the same. A final storage related item worth mentioning is that use of some of the volume types is controlled by a feature gate. E.g. in order to use hostDisk the HostDisk feature gate must be enabled. It is not enabled by default since managing hostDisks would be cumbersome in production, although as mentioned previously they are convenient for development. Enabling KubeVirt feature gates can be done by editing or creating a KubeVirt-config configmap resource specifying HostDisk in the list of 'feature-gates, e.g.

    kubectl create configmap KubeVirt-config -n KubeVirt --from-literal feature-gates=HostDisk

Those with exploration experience know that problems always arise on their journeys, and having tools to solve those problems are paramount for success. When problems arise in Kubernetes, there are many places to look for clues. Along with the usual system logs from cluster nodes, there are logs from the kubelet service running on each node, manifests and state information of cluster resources, logs from cluster pods hosting the resources, and internal pod inspection to name a few. Throughout this journey the most useful sources of information were resource manifests (kubectl get resource resource-name -o yaml), resource current state (kubectl describe resource resource-name), and logs from the pod hosting the resource (kubectl logs resource-pod-name). Inspecting the internal state of a pod was also quite useful at times and can be done by accessing a shell inside the pod (kubectl exec -it pod-name -- /bin/bash).


# Summary

With SUSE's commitment to containerization and the rapid adoption of Kubernetes throughout the IT industry it is desirable to run traditional virtualized workload on Kubernetes clusters. User reap the benefit of using the same platform and tools to run their containerized and virtualized workloads. During this journey it was discovered that KubeVirt extends a Kubernetes cluster capabilities with virtualization using standard Kubernetes APIs, allowing virtualized workloads to run alongside containerized ones using the same tools. KubeVirt enjoys a robust upstream community that is currently discussing the requirements to achieve a "1.0" release. Although note quite as mature as traditional virtualization management tools like libvirt, KubeVirt is rapidly evolving and is worthy of more exploration and commitment from SUSE.


# References

[1] KubeVirt API: https://KubeVirt.io/api-reference/master/index.html

[2] My CaaSP/KubeVirt howto: https://confluence.suse.com/display/virtteam/Installing+KubeVirt+on+CaaSP

[5] Virtlet: https://github.com/Mirantis/virtlet

KubeVirt architecture: https://github.com/KubeVirt/KubeVirt/blob/master/docs/architecture.md

  KubeVirt component descriptions: https://github.com/KubeVirt/KubeVirt/blob/master/docs/components.md
  
  kubernetes CRD: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/
  
  Installing KubeVirt: https://KubeVirtlegacy.gitbook.io/user-guide/installation/installation
  
  [4] Kubernetes Deployment resource: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
  
  KubeVirt google group: https://groups.google.com/forum/#!forum/KubeVirt-dev
  
  KubeVirt slack channel: https://kubernetes.slack.com/messages/virtualization
  
  Leap 15.2-based build container: https://build.opensuse.org/package/show/home:jfehlig:branches:openSUSE:Templates:Images:15.2/KubeVirt-builder
  
  kubectl cheet sheet: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
  
  KubeVirt labs, e.g.: https://KubeVirt.io/labs/kubernetes/lab1
  
  KubeVirt user guide: https://KubeVirt.io/user-guide/#/
  
  [3] KubeVirt user guide Disks and Volumes: https://KubeVirt.io/user-guide/#/
  
  kubernetes PV and PVC: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
