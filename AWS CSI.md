# AWS CSI Driver

## Problems with current in-tree cloud provider

### Cache of used / free device names
On AWS, it's the client who must assign device names to volumes when calling `AWS.AttachVolume`. At the same time, AWS [imposes some restrictions on the device names](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/device_naming.html).

Therefore Kubernetes AWS volume plugin maintains cache of used / free device names for each node. This cache is lost when controller-manager process restarts. We try to populate the cache during startup, however there are some corner cases when this fails. TODO: exact flow how we can get wrong cache.

It would be great if either AWS itself assigned the device names, or there would be robust way how to restore the cache after restart, e.g. using some persistent database. Kubernetes should not care about the device names at all.

### DescribeVolumes quota
In order to attach/detach volume to/from a node, current AWS cloud provider issues `AWS.AttachVolume`/`DetachVolume` call and then it polls `DescribeVolume` until the volume is attached or detached. The frequency of `DescribeVolume` is quite high to minimize delay between AWS finishing attachment of the volume and Kubernetes discovering that. Sometimes we even hit API quota for these calls. 

It would be better if CSI driver could get reliable and fast event from AWS when a volume has become attached / detached.

Or the driver could batch the calls and issue one big `DescribeVolume` call with every volume that's being attached/detached in it.

### AWS API weirdness
AWS API is quite different to all other clouds

* `AWS.AttachVolume`/`DetachVolume` can take ages (days, weeks) to complete. For example, when Kubernetes tries to detach a volume that's still mounted on a node, it will be Detaching until the volume is unmounted or Kubernetes issues force-detach. All other clouds return sane error relatively quickly (e.g. "volume is in use") or force-detach the volume.

* `AWS.DetachVolume` with force-detach is useless. Documentation says:
   >Forced detachment of a stuck volume can cause damage to the file system or the data it contains or an inability to attach a new volume using the same device name, unless you reboot the instance.
   
   We can't reboot instance after each force-detach nor we can afford to "loose" a device name. AWS supports only 40 volumes per node and ever that is quite low number already.

* `AWS.CreateVolume` is not idempotent. There is no way how to create a volume with ID provided by user. Such call would then fail when such volume already exists.

* `AWS.CreateVolume` does not return errors when creating an encrypted volume using either non-existing or non-available KMS key (e.g. with wrong permission). It returns success instead and it event returns volumeID of some volume. This volume exists for a short while and it's deleted in couple of seconds.

### Errors with slow kubelet
Very rarely a node gets too busy and kubelet starves for CPU. It does not unmount a volume when it should and Kubernetes initiates detach of the volume.

## Requirements
### Idempotency
All CSI driver calls should be idempotent. A CSI method call with the same parameters must always return the same result. It's task of CSI driver to ensure that, especially when AWS itself is not idempotent.

Examples:

* `CreateVolume` call must first check that the requested EBS volume has been already provisioned and return it if so. It should create a new volume only when such volume  does not exist.
* `ControllerPublish` (=i.e. attach) does not do anything and returns "success" when given volume is already attached to requested node.
* `DeleteVolume` does not do anything and returns success when given volume is already deleted (i.e. it does not exist, we don't need to check that it had existed and someone really deleted it...)

### Timeouts
gRPC always passes a timeout together with a request. After this timeout, the gRPC client call actually returns. The server (=CSI driver) can continue processing the call and finish the operation, however it has no means how to inform the client about the result.

Kubernetes will  retry failed calls, usually after some exponential backoff. Kubernetes heavily relies on idempotency here - i.e. when the driver finished an operation after the client timed out, the driver will get the same call again and it should return success/error based on success/failure of the previous operation.

Example:

1. Kubernetes calls `ControllerPublishVolume(vol1, nodeA)` ), i.e. "attach vol1 to nodeA".
2. The CSI driver checks vol1 and sees it's not attached to nodeA yet. It calls `AWS.AttachVolume(vol1, nodeA)`.
3. The attachment takes a long time, Kubernetes times out.
4. Kubernetes sleeps for some time.
5. AWS finishes attaching of the volume.
6. Kubernetes re-issues `ControllerPublishVolume(vol1, nodeA)` again.
7. The CSI driver checks vol1 and sees it is attached to nodeA and returns success immediately.

Note that there are some issues:

* Kubernetes can change its mind at any time. E.g. user that wanted to run a pod on the node in the example got impatient so he deleted the pod. In this case Kubernetes calls `ControllerUnpublishVolume(vol1, nodeA)` to "cancel" the attachment request. It's up to the driver to do the right thing - e.g. wait until the volume is attached and then issue `detach()` and wait until the volume is detached and only after that return from `ControllerUnpublishVolume(vol1, nodeA)`.

  Note that Kubernetes may time out waiting for `ControllerUnpublishVolume` too. In this case, it will keep calling it until it gets confirmation from the driver that the volume has been detached (i.e. until the driver returns either success or non-timeout error) or it needs the volume attached to the node again (and it will call `ControllerPublishVolume` in that case).

* The same applies to NodeStage and NodePublish calls ("mount device, mount volume"). These are typically much faster than attach/detach, still they must be idempotent when it comes to timeouts.

*It looks complicated, but it should be actually simple - always check that if the required operation has been already done

### Restarts
The CSI driver should survive its own crashes or crashes or reboots of the node where it runs. For the controller service, Kubernetes will either start a new driver on a different node or re-elect a new leader of stand-by drivers. For the node service, Kubernetes will start a new driver shortly.

The perfect CSI driver should be stateless. After start, it should recover its state by observing the actual status of AWS (i.e. describe instances / volumes). Current cloud provider follows this approach, however there are some corner cases around restarts when Kubernetes can try to attach two volumes to the same device on a node.

When the stateless driver is not possible, it can use some persistent storage outside of the driver. Since the driver should support multiple Container Orchestrators (like Mesos), it must not use Kubernetes APIs. It should use AWS APIs instead to persist its state if needed (like AWS DynamoDB). We assume that costs of using such db will be negligible compared to rest of Kubernetes.

### No credentials on nodes
General security requirements we follow in Kubernetes is "if a node gets compromised then the damage is limited to the node". Paranoid people typically dedicate handful of nodes in Kubernetes cluster as "infrastructure nodes" and dedicate these nodes to run "infrastructure pods" only. Regular users can't run their pods there. CSI attacher and provisioner is an example of such "infrastructure pod" - it need permission to create/delete any PV in Kubernetes and CSI driver running there needs credentials to create/delete volumes in AWS.

There should be a way how to run the CSI driver (=container) in "node mode" only. Such driver would then respond only to node service RPCs and it would not have any credentials to AWS (or very limited credentials, e.g. only to Describe things). Paranoid people would deploy CSI driver in "node only" mode on all nodes where Kubernetes runs user containers.

## Identity Service RPC

### GetPluginInfo
Blindly return:
```yaml
response:
  name: com.aws.csi.ebs # (or a configured value)
  vendor_version: 0.0.1  
```

### GetPluginCapabilities
Blindly return:
```yaml
response:
   capabilities:
     - service:
       type: CONTROLLER_SERVICE
```
### Probe
* Check that the  driver is configured and it can do simple AWS operations, e.g. describe volumes or so.
* This call is used by Kubernetes liveness probe to check that  the driver is healthy. It's called every ~10 seconds (configurable, we can recommend higher values).


## Controller Service RPC
### CreateVolume
Checks that the requested volume was not created yet and creates it.

* Idempotency: several calls with the same `name` parameter must return **the same volume**. We can store this `name` in volume tags in case the driver crashes after `AWS.CreateVolume` call and before returning a response. In other words:
	* The driver first looks for an existing volume with tag `CSIVolumeName=<name>`. It returns it if it's found.
	* When such volume is not found, it calls `AWS.CreateVolume()` to create  the required volume with tag `CSIVolumeName=<name>`.
	* *Is this robust enough? Can this happen on AWS?*
		1. A driver calls `AWS.CreateVolume()` and dies before the new volume is created.
		2. New driver quickly starts, gets the same `driver.CreateVolume` call, checks that there is no volume with given tag (previous `AWS.CreateVolume()` from step 1. has not finished yet) and issues a new `AWS.CreateVolume()`.
		3. Both `AWS.CreateVolume()` calls succeed -> the driver has provisioned 2 volumes for one  `driver.CreateVolume` call.

### DeleteVolume
Checks if the required volume exists and deletes it if so. Returns success if the volume can't be found.

### ControllerPublishVolume
Checks that given volume is already attached to given node. Returns success if so.
Chooses the right device name for the volume on the node (more on that below) and issues `AWS.AttachVolume`.
TODO: this has complicated idempotency expectations. It cancels previously called `ControllerUnpublishVolume` that may be still in progress (i.e. AWS is still detaching the volume and Kubernetes now wants the volume to be attached back).

### ControllerUnpublishVolume
Checks that given volume is not attached to given node. Returns success if so.
Issues `AWS.AttachVolume` and marks the detached device name as free  (more on that below).
TODO: this has complicated idempotency expectations. It cancels previously called `ControllerPublishVolume` (i.e.AWS is still detaching the volume and Kubernetes now wants the volume to be detached).

### ValidateVolumeCapabilities
TBD

### ListVolumes
Not implemented in the initial release, Kubernetes does not need it.

### GetCapacity
Not implemented in the initial release, Kubernetes does not need it.

### ControllerGetCapabilities
Blindly return:
```yaml
response:
  rpc:
    - CREATE_DELETE_VOLUME
    - PUBLISH_UNPUBLISH_VOLUME
```

## Node Service RPC
### NodeStageVolume
1. Find the device.
2. Check if it's unformatted (`lsblk` or `blkid`).
3. Format it if it's needed.
4. `fsck` it if it's already formatted + refuse to mount it on errors.
5. Mount it to given directory.

Steps 3 and 4 can take some times, so the driver must ensure idempotency somehow.

### NodeUnstageVolume
Just unmount the volume.

### NodePublishVolume
Just bind-mount the volume.

### NodeUnpublishVolume
Just unmount the volume.

### NodeGetId
Return AWS InstanceID.

### NodeGetCapabilities
Blindly return:
```yaml
response:
  rpc:
    - STAGE_UNSTAGE_VOLUME
```

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTM0NzY2NDYzLDIwMjU0MzQ4NDVdfQ==
-->