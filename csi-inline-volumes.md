
# In-line CSI volumes in Pods

Author: @jsafrane

## Goal
* Define API and high level design for in-line CSI volumes in Pod

## Motivation
Currently, CSI can be used only though PersistentVolume object. All other persistent volume sources support in-line volumes in Pods, CSI should be no exception. There are three main drivers:
* We want to move away from in-tree volume plugins to CSI, as designed in a separate proposal https://github.com/kubernetes/community/pull/2199/. In-line volumes should use CSI too.
* CSI drivers can be used to provide ephemeral volumes used to inject state, configuration, secrets, identity or similar information to pods, like Secrets and ConfigMap in-tree volumes do today.
	* We don't want to force users to create PVs for each such volume, we should allow to use them in-line in pods as regular Secrets or ephemeral Flex volumes.
	* On the other hand, we expect that CSI volumes used in PVs represent real persistent storage volume, such as iSCSI LUNs or NFS shares.
* Get the same features as Flex and deprecate Flex. I.e. replace it with some CSI-Flex bridge, which is out of scope of this proposal.

## API
`VolumeSource` needs to be extended with CSI volume source:
```go
type VolumeSource struct {
    // <snip>

	// CSI (Container Storage Interface) represents storage that handled by an external CSI driver (Beta feature).
	// +optional
	CSI *CSIVolumeSource
}


// Represents storage that is managed by an external CSI volume driver (Beta feature)
type CSIVolumeSource struct {
	// Driver is the name of the driver to use for this volume.
	// Required.
	Driver string

	// VolumeHandle is the unique ID of the volume. It is the volume ID used in
	// all CSI calls, optionally with a prefix based on VolumeHandlePrefix
	// value.
	// Required
	VolumeHandle string

	// VolumeHandlePrefix is type of prefix added to VolumeHandle before using
	// it as CSI volume ID. It ensures that volumes with the same VolumeHandle
	// in different pods or namespaces get unique CSI volume ID.
	// Required.
	VolumeHandlePrefix CSIVolumeHandlePrefix

	// Optional: The value to pass to ControllerPublishVolumeRequest.
	// Defaults to false (read/write).
	// +optional
	ReadOnly bool

	// Filesystem type to mount.
	// Must be a filesystem type supported by the host operating system.
	// Ex. "ext4", "xfs", "ntfs". Implicitly inferred to be "ext4" if unspecified.
	// +optional
	FSType string

	// Attributes of the volume. This corresponds to "volume_attributes" in some
	// CSI calls.
	// +optional
	VolumeAttributes map[string]string

	// ControllerPublishSecretRef is a reference to the secret object containing
	// sensitive information to pass to the CSI driver to complete the CSI
	// ControllerPublishVolume and ControllerUnpublishVolume calls.
	// This field is optional, and  may be empty if no secret is required. If the
	// secret object contains more than one secret, all secrets are passed.
	// +optional
	ControllerPublishSecretRef *LocalObjectReference

	// NodeStageSecretRef is a reference to the secret object containing sensitive
	// information to pass to the CSI driver to complete the CSI NodeStageVolume
	// and NodeStageVolume and NodeUnstageVolume calls.
	// This field is optional, and  may be empty if no secret is required. If the
	// secret object contains more than one secret, all secrets are passed.
	// +optional
	NodeStageSecretRef *LocalObjectReference

	// NodePublishSecretRef is a reference to the secret object containing
	// sensitive information to pass to the CSI driver to complete the CSI
	// NodePublishVolume and NodeUnpublishVolume calls.
	// This field is optional, and  may be empty if no secret is required. If the
	// secret object contains more than one secret, all secrets are passed.
	// +optional
	NodePublishSecretRef *LocalObjectReference
}

type CSIVolumeHandlePrefix string;
const (
	// VolumeHandle is prefixed by Pod UID.
	CSIVolumeHandlePrefixPod CSIVolumeHandlePrefix  = "Pod"
	// VolumeHandle is prefixed by UID of the namespace where the pod is located.
	CSIVolumeHandlePrefixNamespace CSIVolumeHandlePrefix  = "Namespace"
	// VolumeHandle is not modified.
	CSIVolumeHandlePrefixNone CSIVolumeHandlePrefix = "None"
)
```

The difference between `CSIVolumeSource` (in-lined in a pod) and `CSIPersistentVolumeSource` (in PV) are:

* All secret references in in-line volumes can refer only to secrets in the same namespace where the corresponding pod is running. This is common in all other volume sources that refer to secrets, incl. Flex.
* VolumeHandle in in-line volumes can have a prefix. This prefix (Pod UID, Namespace UID or nothing) is added to the VolumeHandle before each CSI call. It makes sure that each pod uses a different volume ID for its ephemeral volumes.  The prefix must be explicitly set by pod author, there is no default.
	* Each pod created by ReplicaSet, StatefulSet or DaemonSet will get the same copy of a pod template. `CSIVolumeHandlePrefixPod` makes sure that each pod gets its own unique volume ID and thus can get its own volume instance.
	* Without the prefix, user could guess volume ID of a secret-like CSI volume of another user and craft a pod with in-line volume referencing it. CSI driver, obeying idempotency, must then give the same volume to this pod. If users can use only`CSIVolumeHandlePrefixNamespace` or `CSIVolumeHandlePrefixPod`in their in-line volumes, we can make sure that they can't steal secrets of each other.
		* PodSecurityPolicy will be extended to allow / deny users using in-line volumes with no prefix.
	* Finally, `CSIVolumeHandlePrefixNone` allows selected users (configured by their PSP)  to use persistent storage volumes in-line in pods.

## Implementation
#### Provisioning/Deletion
N/A, it works only with PVs and not with in-line volumes.

### Attach/Detach
Current `storage.VolumeAttachment` object contains only reference to PV that's being attached. It must be extended with VolumeSource for in-line volumes in pods.

```go
// VolumeAttachmentSpec is the specification of a VolumeAttachment request.
type VolumeAttachmentSpec struct {
    // <snip>

	// Source represents the volume that should be attached.
	Source VolumeAttachmentSource
}

// VolumeAttachmentSource represents a volume that should be attached, either
// PersistentVolume or a volume in-lined in a Pod.
// Exactly one member can be set.
type VolumeAttachmentSource struct {
	// Name of the persistent volume to attach.
	// +optional
	PersistentVolumeName *string

	// InlineVolumeSource represents the source location of a in-line volume in a pod to attach.
	// +optional
    InlineVolumeSource *InlineVolumeSource
}

// InlineVolumeSource represents the source location of a in-line volume in a pod.
type InlineVolumeSource struct {
	// VolumeSource is copied from the pod. It ensures that attacher has enough
	// information to detach a volume when the pod is deleted before detaching.
	// Only one member can be set.
	// Required.
	VolumeSource v1.VolumeSource

	// Namespace of the pod with in-line volume. It is used to resolve
	// references to Se
	// 
	// Required.
	Namespace string
}
```

* A/D controller **copies whole `VolumeSource`**  from `Pod` into `VolumeAttachment`. This allows external CSI attacher to detach volumes for deleted pods without keeping any internal database of attached VolumeSources.
* Using whole `VolumeSource` allows us to re-use `VolumeAttachment` for any other in-line volume in the future. We provide validation that this `VolumeSource` contains only `CSIVolumeSource` to clearly state that only CSI is supported now.
* External CSI attacher must be extended to  process either `PersistentVolumeName` or `VolumeSource`.
* Since in-line volume in a pod can refer to a secret in the same namespace as the pod, **external attacher may need permissions to read any Secrets in any namespace**.
* CSI `ControllerUnpublishVolume` call (~ volume detach) requires the Secrets to be available at detach time. Current CSI attacher implementation simply expects that the Secrets are available at detach time. Secrets for PVs are "global", out of user's namespace, so this assumption is probably OK. For in-line volumes, **we can either expect that the Secrets are available too (and volume is not detached if user deletes them) or external attacher must cache them somewhere, probably directly in `VolumeAttachment` object itself.**
	* None of existing Kubernetes volume plugins needed credentials for `Detach`, however those that needed it for `TearDown` either required the Secret to be present (e.g. ScaleIO and StorageOS) or stored them in a json in `/var/lib/kubelet/plugins/<plugin name>/<volume name>/file.json` (e.g. iSCSI).

### Kubelet (MountDevice/SetUp/TearDown/UnmountDevice)
In-tree CSI volume plugin calls in kubelet get universal `volume.Spec`, which contains either `v1.VolumeSource` from Pod (for in-line volumes) or `v1.PersistentVolume`. We need to modify CSI volume plugin to check for presence of `VolumeSource` or `PersistentVolume` and read NodeStage/NodePublish secrets from appropriate source. Kubelet does not need any new permissions, it already can read secrets for pods that it handles. These secrets are needed only for `MountDevice/SetUp` calls and don't need to be cached until `TearDown`/`UnmountDevice`.


### Security considerations

* As written above, external attacher may requrie permissions to read Secrets in any namespace. It is up to CSI driver author to document if the driver needs such permission (i.e. access to Secrets at attach/detach time) and up to cluster admin to deploy the driver with these permissions or restrict external attacher to access secrets only in some namespaces.
* PodSecurityPolicy must be enhanced to limit pods in using in-line CSI volumes. It will be modeled following existing Flex volume policy:
  ```go
  type PodSecurityPolicySpec struct {
	// <snip>

	// AllowedFlexVolumes is a whitelist of allowed Flexvolumes.  Empty or nil indicates that all
	// Flexvolumes may be used.  This parameter is effective only when the usage of the Flexvolumes
	// is allowed in the "Volumes" field.
	// +optional
	AllowedFlexVolumes []AllowedFlexVolume

	// AllowedCSIVolumes is a whitelist of allowed CSI volumes.  Empty or nil indicates that all
	// CSI volumes may be used.  This parameter is effective only when the usage of the CSI volumes
	// is allowed in the "Volumes" field.
	// +optional
	AllowedCSIVolumes []AllowedCSIVolume
  }

  // AllowedCSIVolume represents a single CSI volume that is allowed to be used.
  type AllowedCSIVolume struct {
	// Driver is the name of the CSI volume driver.
	Driver string
  }
  ```

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTg4ODY4ODc0MiwtMTU0OTI1Mzc4MiwtMT
Q2MTY1MTMzMywtMTgxNTExNzY1NSw5MzEzMTg3NTksLTE4Njc4
MzQ0MjksLTc2OTI3Mjc0NiwzMjQ2MTQ1NjMsNzc4MjgwMDY1LD
gzMzczNTgwMiw2NTU3NzE4MTMsLTUxNjcwNjY1MF19
-->