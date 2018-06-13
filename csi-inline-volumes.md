# In-line CSI volumes in Pods

Author: @jsafrane

## Goal
* Define API and high level design for in-line CSI volumes in Pod

## Motivation
Currently, CSI can be used only though PersistentVolume object. All other persistent volume sources support in-line volumes in Pods, CSI should be no exception. There are two main drivers:
* We need the API and CSI volume plugin ready to move away from in-tree volume plugins to CSI, as designed in a separate proposal https://github.com/kubernetes/community/pull/2199/
* CSI drivers can be used to provide Secrets-like volumes to pods, e.g. reading secrets from a remote vault. We don't want to force users to create PVs for each secret, we should allow to use them in-line in pods as regular Secrets or Secrets-like flex volumes.

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

	// VolumeHandle is the unique volume name returned by the CSI volume
	// plugin’s CreateVolume to refer to the volume on all subsequent calls.
	// Required.
	VolumeHandle string

	// Optional: The value to pass to ControllerPublishVolumeRequest.
	// Defaults to false (read/write).
	// +optional
	ReadOnly bool

	// Filesystem type to mount.
	// Must be a filesystem type supported by the host operating system.
	// Ex. "ext4", "xfs", "ntfs". Implicitly inferred to be "ext4" if unspecified.
	// +optional
	FSType string

	// Attributes of the volume to publish.
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
```

The only difference between `CSIVolumeSource` (in-lined in a pod) and `CSIPersistentVolumeSource` (in PV) are secrets. All secret references in in-line volumes can refer only to secrets in the same namespace where the pod is running. This is common in all other volume sources that refer to secrets, incl. Flex.

We assume that vast majority of in-line volumes won't need any secrets, as that was the same case in Flex volume.

## Implementation
#### Provisioning/Deletion
N/A, it works only with PVs and not in-line volumes.

### Attach/Detach
Current `storage.VolumeAttachment` object contains only reference to PV that's being attached. It must be extended with VolumeSource for in-line volumes in pods.

```go
// VolumeAttachmentSpec is the specification of a VolumeAttachment request.
type VolumeAttachmentSpec struct {
    // <snip>
    
	// Source represents the volume that should be attached.
	Source VolumeAttachmentSource
}

// VolumeAttachmentSource represents a volume that should be attached.
// Right now only PersistenVolumes can be attached via external attacher,
// in future we may allow also inline volumes in pods.
// Exactly one member can be set.
type VolumeAttachmentSource struct {
	// Name of the persistent volume to attach. Exactly one of PersistentVolumeName or VolumeSource must be specified.
	// +optional
	PersistentVolumeName *string 

	// 
    VolumeSource *v1.VolumeSource 
}
```

* Whole `VolumeSource` is **copied** from `Pod` into `VolumeAttachment` by A/D controller. This allows external CSI attacher to detach volumes for deleted pods without keeping any internal database of attached VolumeSources.
* Using whole `VolumeSource` allows us to re-use `VolumeAttachment` for any other in-line volume in the future. We provide validation that this `VolumeSource` contains only `CSIVolumeSource` to clearly state that only CSI is supported now.
* External CSI attacher must be extended to  process either `PersistentVolumeName` or `VolumeSource`. Since the in-line volume in a pod can refer to a secret in the same namespace as the pod, external attacher must get permissions to read any secret in any name

### MountDevice/SetUp/TearDown/UnmountDevice
In-tree CSI volume plugin calls in kubelet get universal `volume.Spec`, which contains either `v1.VolumeSource` from Pod (for in-line volumes) or `v1.PersistentVolume`. We need to modify CSI volume plugin to check for presence of `VolumeSource` or `PersistentVolume` and read NodeStage/NodePublish secrets from appropriate source.

### Out of tree
#### External provisioner
Nothing needed, it works only with PVs
#### External attacher

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU5MTgzMDAzMyw2NTU3NzE4MTMsLTUxNj
cwNjY1MF19
-->