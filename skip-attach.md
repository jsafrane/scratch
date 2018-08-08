---


---

<h1 id="skip-attach-for-non-attachable-csi-volumes">Skip attach for non-attachable CSI volumes</h1>
<p>Author: @jsafrane</p>
<h2 id="goal">Goal</h2>
<ul>
<li>Non-attachable CSI volumes should be skipped in Attach/Detach controller and they should be mounted right away by kubelet.</li>
</ul>
<h2 id="motivation">Motivation</h2>
<p>Currently, CSI requires admin to start external CSI attacher for <strong>all</strong> CSI drivers, including those that don’t implement attach/detach operation (such as NFS or all ephemeral Secrets-like volumes). Kubernetes Attach/Detach controller always creates <code>VolumeAttachment</code> objects for them and always waits until they’re reported as “attached” by external CSI attacher.</p>
<p>We want to skip creation of <code>VolumeAttachment</code> objects in A/D controller for CSI volumes that don’t require 3rd party attach/detach by A/D controller. We want kubelet to mount a non-attachable CSI volume when it’s scheduled to a node (<code>NodeStage</code> + <code>NodePublish</code> in CSI terms), without waiting for the volume to be attached.</p>
<h2 id="dependencies">Dependencies</h2>
<p>In order to skip both A/D controller attaching a volume and kubelet waiting for the attachment, both of them need to know if a particular CSI driver is attachable or not. In this document we expect that proposal #XYZ is implemented and both A/D controller and kubelet has informer on XYZ so they can check if a volume is attachable easily.</p>
<h2 id="design">Design</h2>
<h3 id="volume-plugins">Volume plugins</h3>
<h4 id="all-volume-plugins">All volume plugins</h4>
<ul>
<li>Extend <code>Attacher</code> interface with <code>IsVolumeAttachable</code> function<pre class=" language-go"><code class="prism  language-go"><span class="token keyword">type</span> Attacher <span class="token keyword">interface</span> <span class="token punctuation">{</span>
    <span class="token comment">// ...</span>
    <span class="token comment">// Checks if the volume specified by the given spec requires attach/detach operation.</span>
    <span class="token comment">// Error is interpreted as the volume is not ready to answer this question yet</span>
    <span class="token comment">// and caller will retry later.</span>
    <span class="token function">IsVolumeAttachable</span><span class="token punctuation">(</span>spec <span class="token operator">*</span>Spec<span class="token punctuation">)</span> <span class="token punctuation">(</span><span class="token builtin">bool</span><span class="token punctuation">,</span> <span class="token builtin">error</span><span class="token punctuation">)</span>
<span class="token punctuation">}</span>
</code></pre>
<ul>
<li>For most volume plugins that implement Attacher interface this will blindly return <code>true</code>.</li>
<li>CSI volume plugin changes are covered below.</li>
</ul>
</li>
</ul>
<h4 id="csi-volume-plugin">CSI volume plugin</h4>
<ul>
<li>Rework <a href="https://github.com/kubernetes/kubernetes/blob/43f805b7bdda7a5b491d34611f85c249a63d7f97/pkg/volume/csi/csi_plugin.go#L58"><code>ProbeVolumePlugins</code></a> to accept informer that watches XYZ. The plugin will store the informer for later.</li>
<li>Implement new <code>IsVolumeAttachable</code> function so it checks the informer and:
<ul>
<li>Return <code>true</code> if the CSI driver is attachable.</li>
<li>Return <code>false</code> otherwise.</li>
<li>Return error if XYZ does not exist yet. We expect that caller (A/D controller or VolumeManager) will retry shortly. This error should be sent to Pod as an event so user knows what Kubernetes is waiting for.</li>
</ul>
</li>
</ul>
<h3 id="ad-controller">A/D controller</h3>
<ul>
<li>A/D controller must pass a (shared) informer to watch XYZ and pass it to CSI volume plugin in <a href="https://github.com/kubernetes/kubernetes/blob/8db5328c4c1f9467ab0d70ccb991a12d4675b6a7/cmd/kube-controller-manager/app/plugins.go#L82"><code>ProbeVolumePlugins</code></a>.</li>
<li>We need to extend the check in <a href="https://github.com/kubernetes/kubernetes/blob/2bd91dda64b857ed2f45542a7aae42f855e931d1/pkg/controller/volume/attachdetach/util/util.go#L209"><code>ProcessPodVolumes</code></a> to call also <code>Attacher.IsVolumeAttachable</code> to distinguish between attachable and non-attachable CSI drivers. Non-attachable volumes will not reach A/D controller’s actual state of the world and will be ignored by the controller.
<ul>
<li>When <code>IsVolumeAttachable</code> returns error (XYZ for the driver does not exist yet), <code>desiredStateOfWorldPopulator</code> seems to retry periodically.</li>
</ul>
</li>
</ul>
<h3 id="kubelet--volumemanager">Kubelet / VolumeManager</h3>
<ul>
<li>Kubelet must create a (shared) informer to watch XYZ and pass it to CSI volume plugin in <a href="https://github.com/kubernetes/kubernetes/blob/8db5328c4c1f9467ab0d70ccb991a12d4675b6a7/cmd/kubelet/app/plugins.go#L101"><code>ProbeVolumePlugins</code></a>.</li>
<li>In VolumeManager, <a href="https://github.com/kubernetes/kubernetes/blob/05c88cc83179449a03e349252cc8ad62a0344109/pkg/kubelet/volumemanager/cache/desired_state_of_world.go#L362-L373"><code>desiredStateOfWorld.isAttachableVolume</code></a> already checks if the volume plugin itself implements <code>AttachableVolumePlugin</code> interface. We need to extend the check to call also <code>Attacher.IsVolumeAttachable</code> to distinguish between attachable and non-attachable CSI drivers.
<ul>
<li>When <code>IsVolumeAttachable</code> returns error (XYZ for the driver does not exist yet), <code>desiredStateOfWorldPopulator</code> seems to retry periodically.</li>
</ul>
</li>
</ul>
<h2 id="api">API</h2>
<p>No API changes</p>
<h2 id="upgrade">Upgrade</h2>
<p>This chapter covers both:</p>
<ul>
<li>Upgrade from old Kubernetes that has <code>CSISkipAttach</code> disabled to new Kubernetes with <code>CSISkipAttach</code> enabled.</li>
<li>Update from Kubernetes that has <code>CSISkipAttach</code> disabled to the same Kubernetes with <code>CSISkipAttach</code> enabled.</li>
</ul>
<p>Admin (or something) must create XYZ instance when "new’ Kubernetes starts. It won’t be able to attach or mount <strong>any</strong> CSI volumes for a driver until XYZ for the CSI drivers is created.</p>
<p>Upgrade does not affect attachable CSI drivers, both “old” and “new” Kubernetes processes them in the same way.</p>
<p>For non-attachable volumes, if the volume was attached by “old” Kubernetes, it has <code>VolumeAttachment</code> instance. We must make sure that something deletes the instance <strong>after</strong> the volume is detached. How???</p>
<h2 id="downgrade">Downgrade</h2>
<p>This chapter covers both:</p>
<ul>
<li>Downgrade from new Kubernetes that has <code>CSISkipAttach</code> enabled to old Kubernetes with <code>CSISkipAttach</code> disabled.</li>
<li>Update from Kubernetes that has <code>CSISkipAttach</code> enabled to the same Kubernetes with <code>CSISkipAttach</code> disabled.</li>
</ul>
<p>Downgrade does not affect attachable CSI drivers, both “old” and “new” Kubernetes processes them in the same way.</p>
<p>For non-attachable volumes, if the volume was attached by “new” Kubernetes, it has no VolumeAttachment instance. “Old” A/D controller will try to attach the volume using dummy CSI attacher. The volume will keep working, nothing unmounts it in the process until corresponding pod is deleted. Unmount will work in the usual way.</p>
<p><strong>Is it really so simple? There are two options what could happen:</strong></p>
<ol>
<li>VolumeManager adds the volume to <code>node.status.volumesInUse</code> even if it is already mounted. When a pod is deleted, kubelet unmounts its volumes and remove the volumes from <code>volumesInUse</code>. A/D controller continues by “detaching” the volumes, i.e. deleting <code>VolumeAttachment</code> instance(s). Since the CSI volume did not implement attach/detach, this is basically NOOP.</li>
<li>VolumeManager does not add the volume to <code>volumesInUse</code>, because it’s already mounted. When the pod is deleted, kubelet unmounts the volume in the usual way. A/D controller will see that the pod is deleted and it’s not in <code>volumesInUse</code> and can potentially “detach” the volume before kubelet finished unmounting, but since the volume does not implement attach/detach, “detach” just deletes <code>VolumeAttachment</code> and has no impact on the volume.</li>
</ol>
<h2 id="implementation">Implementation</h2>
<p>Expected timeline:</p>
<ul>
<li>Alpha: 1.12 (behind feature gate <code>CSISkipAttach</code>)</li>
<li>Beta: 1.13 (enabled by default)</li>
<li>GA: 1.14</li>
</ul>

