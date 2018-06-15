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
* * Rework iSCSI server image
The iSCSI "server container" does not run any daemon. It only configures iSCSI target in kernel. Kernel can can serve multiple LUNs on one node (one for each test) if following conditions are met:
* It must run with HostNetwork=true to be able to serve LUNs from different containers.
* `targetcli` running in containers should see all fake "block devices" (i.e. plain files with ext2 FS in them) that are exported as LUNs, even if such LUN was exported by a different container on the same node. Therefore the container should copy the file to the host, e.g. `/srv/iscsi` directory and this directory is shared to all containers as HostPath volume.

### Deploy iSCSI and Ceph servers on test startup
If we choose to run iSCSI or Ceph servers in `SynchronizedBeforeSuite`:

* Add `--deploy-storage-servers=ceph,iscsi` parameter to e2e.go
* In `SynchronizedBeforeSuite`, run servers specified on the command line. Create appropriate storage classes for them and pre-provision PVs for them in case they don't support dynamic provisioning.
* Rework the tests not to start the servers and use PVCs instead.

### Run all volume plugin tests
We should run all "universal" volume plugin tests in one suite, incl. Ceph and iSCSI.

These tests need:
* iSCSI and Ceph kernel modules, which are not available on GCI / COS (the usual OS for e2e test). Ubuntu is an option.
* Client utilities present on the OS. These are not available neither on COS or Ubuntu image in e2e tests. We have `MOUNT_CONTAINERS` alpha feature that can be used to run NFS, Gluster, iSCSI, Ceph RBD and CephFS mount utilities in containers and not on the host. As benefit, we check that MountPropagation feature works as expected and we catch regressions early.

#### Design
* Prepare a container image with mount utilities for NFS, Gluster, iSCSI, Ceph RBD and CephFS. There is proof-of-concept in [jsafrane/mounter-daemonset repo](https://github.com/jsafrane/mounter-daemonset)

* Add `--deploy-storage-utilities` parameters to `test/e2e.go`. This will cause E2E test to install a DaemonSet with the aforementioned container on all nodes. All nodes then can use NFS, Gluster, iSCSI, Ceph RBD and CephFS (assuming they have correct kernel modules available).

* Prepare new container image with Ceph (RBD + CephFS) server so it supports dynamic provisioning.
	* We want to test Ceph RBD dynamic provisioning.
	* We want to pre-provision number of CephFS PVs so we can run CephFS in parallel (it does not support dynamic provisioning).

*   Add new e2e job (`ci-kubernetes-volumes`) that:
	* Installs Ubuntu (unlike COS it has all the kernel modules)
	* Runs Kubernetes cluster with feature `MountContainers=true`
	* Start e2e tests with `--deploy-storage-utilities`, `--focus=”Feature:Volumes|sig-storage”` and 
`--deploy-storage-servers=iscsi,ceph` (if needed).
	* All iSCSI + NFS + Gluster + Ceph tests will run there (see below how to focus them).
	
* Optionally, add `ci-kubernetes-volumes-serial` to run run all disruptive or serial volume plugin tests.

### Re-tag volume tests
In order to have a new test job that tests most of the volume plugins, it's necessary to add `[Volume:<plugin name>]` to all tests, so we can `--focus=[Volume:.*]` in the test job. Existing `[Feature:Volumes]` will be kept to skip Ceph and iSCSI tests in most jobs that can't run them.


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
eyJoaXN0b3J5IjpbMzA3NDQ3MjYsLTE5MTcwMDg5MjQsMTA5Mj
k3ODgwNl19
-->