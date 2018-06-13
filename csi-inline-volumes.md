# In-line CSI volumes in Pods

Author: @jsafrane

## Goal
* Define API and high level design for in-line CSI volumes in Pod

## Motivation
Currently, CSI can be used only though PersistentVolume object. All other persistent volume sources support in-line volumes in Pods, CSI should be no exception. There are two main drivers:
* We need the API and CSI volume plugin ready to move away from in-tree volume plugins to CSI, as designed in a separate proposal https://github.com/kubernetes/community/pull/2199/
* CSI drivers can be used to provide non-persistent storage, for example Secrets-like volumes to pods, e.g. reading secrets from a remote vault. We don't want to force users 
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE4NjI4NjUwMTcsNjU1NzcxODEzLC01MT
Y3MDY2NTBdfQ==
-->