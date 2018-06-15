# Kubernetes volume plugin tests
## Goal
* Capture current state of e2e tests of Kubernetes volume plugins.
* Outline new e2e jobs to test volume plugins.

### Out of scope
* Restructure tests so all storage features are tested with as much volume plugins as possible.

## Current volume plugin tests
E2e tests are in `test/e2e/storage`.

Current volume plugins:

### Cloud based volume plugins
  
Existing volume tests:
 
|Volume plugin | Dynamic provisioning tests in `volume_provisioning.go` | In-line volume in pod in `volumes.go` | Additional feature tests | Cloudprovider specific tests
|--|--|--|--|--|
| AWS EBS         | yes | yes | Resize, Attach/Detach (`pd.go`), Metrics |
| Azure DD        | yes | yes | **none** |
| Azure File      | **no** | **no** | **none**
| GCE PD          | yes | yes | Resize, Attach/Detach (`pd.go`), Subpath, Metrics, MountOptions | Regional PD, CSI 
| OpenStack Cinder| yes | yes | **none**
| vSphere disk    | yes | yes | **none** | whole `vsphere/` subdirectory 
 
 These tests have correct `framework.SkipUnlessProviderIs("<cloud>")`, so they can run if Kubernetes had corresponding e2e job in appropriate cloud. Kubernetes runs these jobs:

|Volume plugin | Test jobs | Comments
|--|--|--|
| AWS EBS         | `pull-kubernetes-e2e-kops-aws` | Does not cover `volume_provisioning.go` because of `[Slow]`
| Azure DD        | **no** |
| GCE PD          | `pull-kubernetes-e2e-gce` and number of others (`-slow`, `-serial`, `-disruptive`)
| OpenStack Cinder| **no** |
| vSphere disk    | **no** | VMware runs their own e2e


### Universal volume plugins
These can run anywhere, given that the platform provides kernel modules, server(s) and client utilities.

#### Test coverage
| Volume plugin | Dynamic provisioning tests in `volume_provisioning.go` | In-line volume in pod in `volumes.go` | Additional feature tests | Plugin specific tests
|--|--|--|--|--|
| ConfigMap, DownwardAPI, Projected, Secrets| N/A | N/A | Subpath, FSGroup | `test/e2e/common/*.go`
| CephFS | N/A | Yes |
| CSI | Yes (GCE PD) | No |
| EmptyDir | N/A | No | Subpath |
| FC | N/A | No | | Requires extra HW
| Flex | N/A | No | | Whole `flexvolume.go` (dummy driver?)
| Flocker | N/A | No | | Deprecated?
| Git repo | N/A | No | | `empty_dir_wrapper.go`
| GlusterFS | Yes | Yes | Subpath |
| HostPath | N/A | 
| iSCSI | N/A | Yes |
| Local | Yes | No | | `persistent_volumes-local.go`
| NFS | Yes (via external provisioner) | Yes | Subpath | `nfs_persistent_volume-disruptive.go`
| Photon PD | N/A | No |
| Portworx | No | No |
| Quobyte | No | No |
| Ceph RBD | No | Yes |
| ScaleIO | No | No |
| StorageOS | No | No |

*) Atomic Writer = ConfigMap, DownwardAPI, Projected and Secrets volumes.

#### Test jobs
For those plugins that have some tests, we run them in these test jobs:

|Volume plugin | Test jobs | Test tags | Comments
|--|--|--|--|
| ConfigMap, DownwardAPI, Projected, Secrets| All conformance jobs | None | 
| CephFS | **none** | `[Feature:Volumes]`| Requires Ceph kernel modules and client utilities
| CSI | `pull-kubernetes-e2e-gce` | None? | HostPath dummy only?
| EmptyDir | All conformance jobs | None | 
| Flex | `ci-kubernetes-gci-gce-serial` | `[Disruptive]`
| Git repo | All conformance jobs? | None |
| GlusterFS | `pull-kubernetes-e2e-gce` | None, `SkipUnlessNodeOSDistroIs("gci", "ubuntu")`
| HostPath | All conformance jobs |  None |
| iSCSI | **none** | `[Feature:Volumes]` | Requires iSCSI kernel modules and client utilities | 
| Local | `pull-kubernetes-e2e-gce` | `SkipUnlessProviderIs(ProvidersWithSSH)`
| NFS | `pull-kubernetes-e2e-gce` | None
| Ceph RBD | **none** | `[Feature:Volumes]` | Requires Ceph kernel modules and client utilities

Individual tests have additional `[Slow]`, `[Serial]` and `[Disruptive]` tags as appropriate.
`[Feature:Volumes]` is used in Ceph and iSCSI tests to skip them in all jobs, because no job install Ceph or iSCSI client utilities.

#### Ceph server image
Current Ceph RBD and CephFS tests start a new server in each test. Current Ceph image at  `test/images/volumes-tester/rbd` can run only **once** per node, because the image has hardcoded RBD pool and RBD image ("volume") name.
 
 This should be fixed, we don't want Ceph tests to be `[Serial]`.
 
#### iSCSI server image
Similarly, only one iSCSI container based on   `test/images/volumes-tester/iscsi`, because it configures iSCSI target ("server") in kernel and does not count with multiple such containers configuring the same kernel.

This should be fixed, we don't want iSCSI tests to be `[Serial]`.
 
## Proposed changes

* Remove `[Slow]` from `volume_provisioning.go`. On GCE, it tests 3 storage classes in 46 seconds. We have ~7 storage classes on AWS, it could take 2-3 minutes and it's not that slow.

* Rework Ceph server image to be able to run multiple times on a node.

* Rework iSCSI server image to be able to run multiple times on a node.

* Add a new test job that will run tests for all volume plugins incl. iSCSI and Ceph. This requires multiple changes covered in the chapter below.
 
### Extra job for volume plugin tests.
As written above, iSCSI, Ceph RBD and CephFS test have `[Feature:Volumes]` tag and do not run in any existing job, because no job install Ceph or iSCSI client utilities and/or kernel modules.

In order to run these tests, we need:
* **Prepare a container image with mount utilities for NFS, Gluster, iSCSI, Ceph RBD and CephFS**. There is proof-of-concept in [jsafrane/mounter-daemonset repo](https://github.com/jsafrane/mounter-daemonset). It will end up in `test/images/volume-tester/mount`. Kubelet already has a way howto run these mount utilities in containers instead on host, see below.

* **Add new option `--deploy-storage-utilities` parameters to `test/e2e.go`**. This will cause E2E test to install a DaemonSet with the aforementioned container on all nodes in`SynchronizedBeforeSuite`. All nodes then can use NFS, Gluster, iSCSI, Ceph RBD and CephFS volumes.

* **Create a new job `pull-kubernetes-gce-volumes`** that:
	* Installs Ubuntu cluster with`MountContainers` alpha feature enabled
	  *	Ubuntu is used to get all necessary kernel modules. COS does not ship them.
	  *	`MountContainers` alpha feature [allows kubelet to run mount utilities in containers instead on the host](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/containerized-mounter-pod.md#implementation-notes).
  * Runs the tests with `e2e.test --deploy-storage-utilities` to deploy the mount utilities for kubelet in containers.
  * Runs all storage tests with: `--ginkgo.focus=[sig-storage] --ginkgo.skip=[Distruptive]|[Flaky]|[Serial]|[Feature:<all features except Volumes>]`.
	  * We want `[Feature:Volumes]` in and  all other `[Feature:.*]` out. Go regexp does not allow negative matching `(?!Feature:Volumes)`, so 

I tried all the above with Kubernetes cluster started in this way:
```
KUBE_FEATURE_GATES=MountContainers=true KUBE_GCE_NODE_IMAGE=ubuntu-gke-1604-xenial-v20170816-1 KUBE_GCE_NODE_PROJECT=ubuntu-os-gke-cloud KUBE_NODE_OS_DISTRIBUTION=ubuntu  cluster/kube-up.sh
```
And ran the tests:
```
go run hack/e2e.go  -- --test  --test_args="
    --ginkgo.focus=\\[sig-storage\\]
    --ginkgo.skip=\\[Disruptive\\]|\\[Flaky\\]|\\[Serial\\]|\\[Feature:([^V]|V[^o]|Vo[^l]|Vol[^u]|Volu[^m]|Volum[^e]|Volume[^s]).*\\]|\\[NodeFeature:.+\\]"
```

Notice complicated `--skip` - we want `Feature:Volumes` in and all other features out. It might be better to re-tag `[Feature:Volumes]` to `[Volumes]`, but then it needs to be skipped in all other jobs.

## Future directions
Out of scope of this proposal:
* Refactor tests for individual features so we can test the feature with all volume plugins that support it.
	* Candidates:
		* Mount options
		* FSGroup
		* SELinux (if possible)
		* Resize
		* Attach limits
		* Subpath
	* Subpath is a great example. It already has tests for most volume plugins, we should refactor it into some generic framework.

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTk5NzkxMTc5OCwtMjA2OTgwNTA5MCwtMT
E5MDI5MTU3OSwtMTkyNjg4NTA4MiwtMTE5MTcxMzEwMSwtMTkx
NzAwODkyNCwxMDkyOTc4ODA2XX0=
-->