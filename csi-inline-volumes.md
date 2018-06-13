# In-line CSI volumes in Pods

Author: @jsafrane

## Goal
* Define API and high level design for in-line CSI volumes in Pod

## Motivation
Currently, CSI can be used only though PersistentVolume object. All other persistent volume sources support in-line volumes in Pods, CSI should be no exception. It will be useful when we move away from in-tree drive 
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwODI2MzkzMzIsNjU1NzcxODEzLC01MT
Y3MDY2NTBdfQ==
-->