---


---

<h1 id="local-storage-operator">Local Storage Operator</h1>
<h2 id="overview">Overview</h2>
<p>This is an operator for Kubernetes <a href="https://github.com/kubernetes-incubator/external-storage/tree/master/local-volume">local storage provisioner</a>. It simplifies deployment and configuration of local volumes in Kubernetes clusters.</p>
<h3 id="goals">Goals</h3>
<ul>
<li>Simplify configuration of local storage, especially around discovery of devices to manage.</li>
<li>Allow for update of local storage provisioner image.</li>
</ul>
<h3 id="problems">Problems</h3>
<p>Local storage as implemented in Kubernetes does not provide any means how to discover devices which should be exposed as local PVs on nodes. Admin (or a script) must:</p>
<ol>
<li>Choose which devices should be used on each node.</li>
<li>Choose how the devices are going to be used, either as block devices with <code>volumeMode=Block</code> or as filesystems with <code>volumeMode=Filesystem</code>.</li>
<li>Mount the devices that should be used as filesystems to a specific directory.</li>
<li>Link block devices that  should be used as block devices to the same directory.</li>
</ol>
<p>The specific directory is different for each storage class that the admin wants to use, e.g. <code>/opt/kubernetes/local-ssd</code> for storage class <code>local-ssd</code> or <code>/opt/kubernetes/local-hdd</code> for <code>'local-hdd</code>.</p>
<h2 id="design">Design</h2>
<h3 id="use-block-volumes-for-filesystem-pvs">Use block volumes for filesystem PVs</h3>
<p>Currently, local PVs with <code>volumeMode=Filesystem</code> must be mounted first on the node by something. Kubernetes local volume plugin will <strong>not</strong> create a filesystem nor mount them as part of <code>SetUp</code> call.<br>
All other volume plugins that are based on block devices (iSCSI, FC, AWS EBS, GCE PV, Cinder, Azure, …) do create filesystem and mount its device in <code>SetUp</code> (or <code>MountDevice</code>) calls.<br>
We should update Kubernetes local volume plugin to format and mount the device when a PV with <code>volumeMode=Filesystem</code> is backed by a block device. There is <a href="https://github.com/kubernetes/community/pull/1683">proposal</a> for that, with some open issues. We accumulated some technical debt in this area and reviewers want to clean it up.</p>
<h3 id="api">API</h3>
<p>Local Storage Operator is an external controller watching LocalStorage CRD</p>
<pre class=" language-go"><code class="prism  language-go"><span class="token comment">// LocalStorage is a specification for a local storage deployment</span>
<span class="token keyword">type</span> LocalStorage <span class="token keyword">struct</span> <span class="token punctuation">{</span>
	metav1<span class="token punctuation">.</span>TypeMeta   <span class="token string">`json:",inline"`</span>
	metav1<span class="token punctuation">.</span>ObjectMeta <span class="token string">`json:"metadata,omitempty"`</span>

	Spec   LocalStorageSpec   <span class="token string">`json:"spec"`</span>
	Status LocalStorageStatus <span class="token string">`json:"status"`</span>
<span class="token punctuation">}</span>

<span class="token comment">// LocalStorageSpec is the spec for a LocalStorage resource</span>
<span class="token keyword">type</span> LocalStorageSpec <span class="token keyword">struct</span> <span class="token punctuation">{</span>
	<span class="token comment">// List of storage classes for local storage. These storage classes will be</span>
	<span class="token comment">// automatically created.</span>
	StorageClasses <span class="token punctuation">[</span><span class="token punctuation">]</span><span class="token builtin">string</span> <span class="token string">`json:"storageClasses"`</span>
	<span class="token comment">// List of matching rules for devices on nodes.</span>
	Nodes <span class="token punctuation">[</span><span class="token punctuation">]</span>NodeGroup <span class="token string">`json:"nodes"`</span>
<span class="token punctuation">}</span>

<span class="token comment">// NodeGroup reperesent a group of nodes that have the same device matching</span>
<span class="token comment">// rules.</span>
<span class="token keyword">type</span> NodeGroup <span class="token keyword">struct</span> <span class="token punctuation">{</span>
	<span class="token comment">// Selector that matches all members of the NodeGroup.</span>
	<span class="token comment">// Only one of nodeSelector or nodeName can be set. All nodes are matched</span>
	<span class="token comment">// when neither of them is set.</span>
	<span class="token comment">// +optional</span>
	NodeSelector <span class="token operator">*</span>corev1<span class="token punctuation">.</span>NodeSelector <span class="token string">`json:"nodeSelector"`</span>
	<span class="token comment">// Name of node that composes the group of nodes.</span>
	<span class="token comment">// Only one of nodeSelector or nodeName can be set. All nodes are matched</span>
	<span class="token comment">// when neither of them is set.</span>
	<span class="token comment">// +optional</span>
	NodeName <span class="token operator">*</span><span class="token builtin">string</span>
	<span class="token comment">// List of device matching rules that apply to the NodeGroup.</span>
	Devices <span class="token punctuation">[</span><span class="token punctuation">]</span>DeviceGroup <span class="token string">`json:"devices"`</span>
<span class="token punctuation">}</span>

<span class="token comment">// DeviceGroup represents matching rules for devices that belong to a particular</span>
<span class="token comment">// StorageClass in a NodeGroup.</span>
<span class="token keyword">type</span> DeviceGroup <span class="token keyword">struct</span> <span class="token punctuation">{</span>
	<span class="token comment">// Name of the storage class where matched devices belong.</span>
	StorageClassName <span class="token builtin">string</span> <span class="token string">`json:"storageClassName"`</span>
	<span class="token comment">// Volume mode of matched devices.</span>
	VolumeMode corev1<span class="token punctuation">.</span>PersistentVolumeMode
	<span class="token comment">// List of wildcards that match individual devices (i.e. their kernel names,</span>
	<span class="token comment">// such as "sda", "sd[b-z]" or "dm-45"). It is not possible to match stable</span>
	<span class="token comment">// device names (in /dev/disk/by-id) this way, use udevRules instead.</span>
	DeviceNames <span class="token punctuation">[</span><span class="token punctuation">]</span><span class="token builtin">string</span> <span class="token string">`json:"deviceNames"`</span>
	<span class="token comment">// udev rule that matches the devices. Only devices with</span>
	<span class="token comment">// 'SUBSYSTEM=="block"' will reach these rules. Invalid rules can potentially</span>
	<span class="token comment">// damage the node!</span>
	UdevRules <span class="token punctuation">[</span><span class="token punctuation">]</span><span class="token builtin">string</span> <span class="token string">`json:"udevRules"`</span>
<span class="token punctuation">}</span>
</code></pre>
<h4 id="examples">Examples</h4>
<ul>
<li>Admin wants to match <code>/dev/sdb</code> - <code>/dev/sdz</code> on all nodes as raw block PVs:<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">kind</span><span class="token punctuation">:</span> LocalStorage
<span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> local.storage.openshift.com/v1alpha1
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> local
<span class="token key atrule">spec</span><span class="token punctuation">:</span>
  <span class="token key atrule">nodes</span><span class="token punctuation">:</span>
    <span class="token comment"># No selector implies all nodes</span>
    <span class="token punctuation">-</span> <span class="token key atrule">devices</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token key atrule">storageClassName</span><span class="token punctuation">:</span> local
        <span class="token key atrule">volumeMode</span><span class="token punctuation">:</span> Block
        <span class="token key atrule">deviceNames</span><span class="token punctuation">:</span>
        <span class="token punctuation">-</span> sd<span class="token punctuation">[</span>b<span class="token punctuation">-</span>z<span class="token punctuation">]</span>
</code></pre>
</li>
<li>Admin wants to match all devices except <code>sda</code> on nodes with label <code>localStorage=="true"</code> and have a separate storage class for SSD and HDD.<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">kind</span><span class="token punctuation">:</span> LocalStorage
<span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> local.storage.openshift.com/v1alpha1
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> local
<span class="token key atrule">spec</span><span class="token punctuation">:</span>
  <span class="token key atrule">storageClasses</span><span class="token punctuation">:</span>
    <span class="token punctuation">-</span> local
  <span class="token key atrule">nodes</span><span class="token punctuation">:</span>
    <span class="token punctuation">-</span> <span class="token key atrule">nodeSelector</span><span class="token punctuation">:</span>
        <span class="token key atrule">localStorage</span><span class="token punctuation">:</span> <span class="token string">"true"</span>
      <span class="token key atrule">devices</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token key atrule">storageClassName</span><span class="token punctuation">:</span> local<span class="token punctuation">-</span>ssd
        <span class="token key atrule">volumeMode</span><span class="token punctuation">:</span> Block
        <span class="token key atrule">udevRules</span><span class="token punctuation">:</span>
        <span class="token punctuation">-</span> <span class="token string">"KERNEL!=\"sda\" , ATTR{queue/rotational}==\"0\""</span>
      <span class="token punctuation">-</span> <span class="token key atrule">storageClassName</span><span class="token punctuation">:</span> local<span class="token punctuation">-</span>hdd
        <span class="token key atrule">volumeMode</span><span class="token punctuation">:</span> Block
        <span class="token key atrule">udevRules</span><span class="token punctuation">:</span>
        <span class="token punctuation">-</span> <span class="token string">"KERNEL!=\"sda\",  ATTR{queue/rotational}==\"1\""</span>
</code></pre>
</li>
</ul>
<p>Based on a CR, the operator creates a new DaemonSet with its own ConfigMap(s) for each <code>NodeGroup</code>. The <code>DaemonSet</code> runs two containers, local storage discoverer and local storage provisioner.</p>
<h3 id="local-storage-discoverer">Local storage discoverer</h3>
<p>This is a new component that inserts udev rules to the host. On each change of provided ConfigMap with the udev rules, it creates / overwrites <code>/etc/udev/rules.d/99-kubernetes-local-storage-&lt;LocalStorage.Name&gt;.rules</code> and re-triggers all udev rules.</p>
<p>The udev rules create symlinks to matched devices in <code>/dev/disk/kubernetes-local-storage/&lt;storageClassName&gt;/&lt;volumeMode&gt;/&lt;device ID&gt;</code>, where:<br>
* <code>storageClassName</code> is name of the storage class from <code>DeviceGroup.storageClassName</code>.<br>
* <code>volumeMode</code> is either “block” or “filesystem”, based on <code>DeviceGroup.volumeMode</code>.<br>
* <code>deviceID</code> is <code>$env{ID_BUS}-$env{ID_SERIAL}</code> for the device, when <code>ID_BUS</code> and <code>ID_SERIAL</code> exist (i.e. the same name as is in <code>/dev/disk/by-id/</code>, or <code>$kernel</code>, when <code>ID_BUS</code> or <code>ID_SERIAL</code> does not exist (i.e. the device name, such as <code>sda</code>).</p>
<p>It is expected that local volume provisioner watches over <code>/dev/disk/kubernetes-local-storage/&lt;storageClassName&gt;/&lt;volumeMode&gt;/</code> and creates PVs as new devices appear there.</p>
<p>Sample udev rules file:</p>
<pre><code># Static header
# Process only block files
KERNEL!="block" GOTO="kubernetes_local_storage_end"


# Rules from LocalStorage CRD - we just add KUBERNETES_STORAGE_CLASS and KUBERNETES_VOLUME_MODE env. variables to matched devices.
"KERNEL!="sda" , ATTR{queue/rotational}=="0", ENV{KUBERNETES_STORAGE_CLASS}="local-ssd", ENV{KUBERNETES_VOLUME_MODE}="block"
"KERNEL!="sda" , ATTR{queue/rotational}=="1", ENV{KUBERNETES_STORAGE_CLASS}="local-hdd", ENV{KUBERNETES_VOLUME_MODE}="block"


# Static footer
# The device was not matched, finish processing
ENV{KUBERNETES_STORAGE_CLASS}=="" GOTO="kubernetes_local_storage_end"
# The device matched, create a symlink with stable device name, if possible
ENV{ID_SERIAL}=="?*" SYMLINK+="disk/kubernetes-local-storage/$env{KUBERNETES_VOLUME_MODE}/$env{ID_BUS}-$env{ID_SERIAL}

# Fall back to kernel name if stable device name does not exist
ENV{ID_SERIAL}=="?*" SYMLINK+="disk/kubernetes-local-storage/$env{KUBERNETES_VOLUME_MODE}/$kernel

LABEL="kubernetes_local_storage_end"
</code></pre>
<p>As can be seen, the rules file consists of stable header, list of rules from CR and stable footer. Udev rules are generated also for <code>devices.deviceNames</code> so the handling is consistent.</p>
<p>The discoverer tries to delete <code>/etc/udev/rules.d/99-kubernetes-local-storage-&lt;LocalStorage.Name&gt;.rules</code> upon death. <strong>It may leave orphan rules e.g. when the CR is deleted while node is shut down. We need some cleanup…</strong></p>
<h3 id="local-storage-provisioner">Local storage provisioner</h3>
<p>This component is  developed <a href="https://github.com/kubernetes-incubator/external-storage/tree/master/local-volume">upstream</a>. It watches over configured directories and creates PV for device symlinks it finds there.</p>
<p>The operator must make sure that the right directories are watched.</p>
<p><strong>Note</strong>: Unlike upstream, it’s not possible to use mounted directories as local storage PVs. We support only unmounted local block devices!</p>

