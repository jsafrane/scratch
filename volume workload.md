---


---

<h1 id="pod-in-csi-nodepublish-request">Pod in CSI NodePublish request</h1>
<p>Author: @jsafrane</p>
<h2 id="goal">Goal</h2>
<ul>
<li>Pass Pod information (pod name/namespace + service account) in CSI NodePublish request as volume attributes.</li>
<li>Establish API where admin can register which CSI drivers should receive Pod in NodePublish and which don’t want it.</li>
</ul>
<h2 id="motivation">Motivation</h2>
<p>We’d like to move away from exec based Flex to gRPC based CSI volumes. In Flex, kubelet <strong>always</strong> passes <code>pod.namespace</code>, <code>pod.name</code>, <code>pod.uid</code> and <code>pod.spec.serviceAccountName</code> (“pod information”) in every <code>mount</code> call. In Kubernetes community we’ve seen some Flex drivers that use pod or service account information to authorize or audit usage of a volume or generate content of the volume tailored to the pod (e.g. <a href="https://github.com/Azure/kubernetes-keyvault-flexvol">https://github.com/Azure/kubernetes-keyvault-flexvol</a>).</p>
<p>CSI is agnostic to container orchestrators (such as Kubernetes, Mesos or CloudFoundry) and as such does not understand  concept of pods and service accounts. <a href="https://github.com/container-storage-interface/spec/pull/252">Enhancement of CSI protocol</a> to pass “workload” (~pod) information from Kubernetes to CSI driver has met some of resistance.</p>
<h2 id="high-level-design">High-level design</h2>
<p>We decided to pass the pod information as <code>NodePublishVolumeRequest.volume_attributes</code>.</p>
<ul>
<li>Kubernetes passes pod information only to CSI drivers that explicitly require that information. These drivers are tightly coupled to Kubernetes and may not work or may require reconfiguration on other cloud orchestrators. It is expected (but not limited to) that these drivers will provider ephemeral volumes similar to Secrets or ConfigMap, extending Kubernetes secret or configuration sources.</li>
<li>Kubernetes will not pass pod information to CSI drivers that don’t know or don’t care about pods and service accounts. It is expected (but not limited to) that these drivers will provide real persistent storage. Such CSI driver would reject a CSI call with pod information as invalid.</li>
</ul>
<h2 id="detailed-design">Detailed design</h2>
<h3 id="csi-enhancement">CSI enhancement</h3>
<p>We don’t need to change CSI protocol in any way. It allows kubelet to pass <code>pod.name</code>, <code>pod.uid</code> and <code>pod.spec.serviceAccountName</code> in <a href="(https://github.com/container-storage-interface/spec/blob/master/spec.md#nodepublishvolume)"><code>NodePublish</code> call as <code>volume_attributes</code></a>. <code>NodePublish</code> is roughly equivalent to Flex <code>mount</code> call.</p>
<p>The only thing we need to do is to <strong>define</strong> names of the <code>volume_attributes</code> keys that CSI drivers can expect:<br>
*	<code>csi.kubernetes.io/pod.name</code>: name of the pod that wants the volume.<br>
*	<code>csi.kubernetes.io/pod.namespace</code>: namespace of the pod that wants the volume.<br>
*	<code>csi.kubernetes.io/pod.uid</code>: uid of the pod that wants the volume.<br>
*	<code>csi.kubernetes.io/serviceAccount.name</code>: name of the service account under which the pod operates. Namespace is the same as <code>pod.namespace</code>.</p>
<p>Note that these attribute names are very similar to <a href="https://github.com/kubernetes/kubernetes/blob/10688257e63e4d778c499ba30cddbc8c6219abe9/pkg/volume/flexvolume/driver-call.go#L55">parameters we pass to flex volume plugin</a>. Kubernetes overwrites these <code>volume_attributes</code> if they were already present in a CSI volume (either in a PV or in-line in a pod).</p>
<h3 id="kubernetes">Kubernetes</h3>
<p>Right now, we have a sidecar container called Driver Registrar that runs with a CSI driver container on every node (via DaemonSet). This driver registrar implements <a href="https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/apis/pluginregistration/v1alpha1/api.proto">plugin registration v1alpha1 API</a>  and sends path to a CSI driver UNIX domain socket to kubelet. In kubelet, this socket is passed to <a href="https://github.com/kubernetes/kubernetes/blob/10688257e63e4d778c499ba30cddbc8c6219abe9/pkg/volume/csi/csi_plugin.go#L87">CSI driver plugin</a>, which then keeps track of registered CSI drivers and their sockets.</p>
<p>We need to extend this interface to pass also a flag if the CSI driver wants to receive pod information in <code>NodePublish</code> requests. Such flag can’t be passed via CSI itself, because CSI itself does not know (and refuses to know) anything about pods or service accounts. Therefore we propose a new gRPC API between the CSI Driver Registar and kubelet.</p>
<p>Location: <code>pkg/volumes/apis/csiregistration/v1alpha1/api.proto</code></p>
<pre class=" language-protobuf"><code class="prism  language-protobuf"><span class="token keyword">package</span> csiregistration<span class="token punctuation">;</span>

<span class="token keyword">import</span> <span class="token string">"github.com/gogo/protobuf/gogoproto/gogo.proto"</span><span class="token punctuation">;</span>

<span class="token function">option</span> <span class="token punctuation">(</span>gogoproto<span class="token punctuation">.</span>goproto_stringer_all<span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token boolean">false</span><span class="token punctuation">;</span>
<span class="token function">option</span> <span class="token punctuation">(</span>gogoproto<span class="token punctuation">.</span>stringer_all<span class="token punctuation">)</span> <span class="token operator">=</span>  <span class="token boolean">true</span><span class="token punctuation">;</span>
<span class="token function">option</span> <span class="token punctuation">(</span>gogoproto<span class="token punctuation">.</span>goproto_getters_all<span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token boolean">true</span><span class="token punctuation">;</span>
<span class="token function">option</span> <span class="token punctuation">(</span>gogoproto<span class="token punctuation">.</span>marshaler_all<span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token boolean">true</span><span class="token punctuation">;</span>
<span class="token function">option</span> <span class="token punctuation">(</span>gogoproto<span class="token punctuation">.</span>sizer_all<span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token boolean">true</span><span class="token punctuation">;</span>
<span class="token function">option</span> <span class="token punctuation">(</span>gogoproto<span class="token punctuation">.</span>unmarshaler_all<span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token boolean">true</span><span class="token punctuation">;</span>
<span class="token function">option</span> <span class="token punctuation">(</span>gogoproto<span class="token punctuation">.</span>goproto_unrecognized_all<span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token boolean">false</span><span class="token punctuation">;</span>

<span class="token comment">// CSIDriverInfo is the message sent from a CSI driver registrar to the Kubelet pluginwatcher for CSI driver registration</span>
<span class="token keyword">message</span> CSIDriverInfo <span class="token punctuation">{</span>
        <span class="token comment">// UNIX Socket with CSI driver endpoint</span>
        <span class="token primitive symbol">string</span> csi_endpoint <span class="token operator">=</span> <span class="token number">1</span><span class="token punctuation">;</span>
        <span class="token comment">// Configuration of kubelet behavior for this driver</span>
        CSIDriverOptions options <span class="token operator">=</span> <span class="token number">2</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

<span class="token keyword">message</span> CSIDriverOptions <span class="token punctuation">{</span>
        <span class="token comment">// Indicates if kubelet shall send Pod in NodePublish requests</span>
        <span class="token primitive symbol">bool</span> requires_publish_pod <span class="token operator">=</span> <span class="token number">1</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>


<span class="token comment">// CSIDriverInfo is the empty request message from Kubelet</span>
<span class="token keyword">message</span> CSIDriverInfoRequest <span class="token punctuation">{</span>
<span class="token punctuation">}</span>

<span class="token comment">// Registration is the service advertised by the Plugins.</span>
service CSIRegistration <span class="token punctuation">{</span>
        rpc <span class="token function">GetCSIDriverInfo</span><span class="token punctuation">(</span>CSIDriverInfoRequest<span class="token punctuation">)</span> <span class="token function">returns</span> <span class="token punctuation">(</span>CSIDriverInfo<span class="token punctuation">)</span> <span class="token punctuation">{</span><span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre>
<p>This API will be accompanied with <code>hack/update-generated-csi-registration.sh</code> and <code>hack/update-generated-csi-registration-dockerized.sh</code> to generate golang code for the API definition (based on <code>hack/update-generated-device-plugin*.sh</code>).</p>
<h4 id="csi-volume-plugin">CSI volume plugin</h4>
<ul>
<li>It already has a map of registered CSI drivers and their sockets. This map must be extended with a flag if a driver wants to receive pod information in <code>NodePublish</code>.</li>
<li>It implements Client-side of the <code>CSIRegistration</code> API. When the plugin gets a callback that a new CSI socket (created by Driver Registrar) was detected, it uses this socket to send <code>CSIDriverInfoRequest</code> to the Driver Registrar. The Driver Registrar either returns <code>CSIDriverInfo</code> with the flag or the Driver Registrar does not implement the method and we can assume that the corresponding CSI driver does not want to receive pod information.</li>
<li>Based on the flag above, it passes pod information in <code>NodePublish</code> requests.</li>
</ul>
<h4 id="csi-driverregistrar">CSI DriverRegistrar</h4>
<ul>
<li>It has a new command line option <code>--requires-publish-pod</code> that admin (or CSI driver author) sets if the CSI driver needs pod information in <code>NodePublish</code>.</li>
<li>It implements the server part of <code>CSIRegistration</code> service, i.e. listens for  <code>GetCSIDriverInfo</code> call and return <code>CSIDriverInfo</code> with the flag.</li>
</ul>
<h2 id="end-to-end-examples">End to end examples</h2>
<h3 id="csi-driver-wants-pod-information-in-nodepublish">CSI driver wants pod information in <code>NodePublish</code></h3>
<ol>
<li>CSI driver author provides documentation and templates that the driver need  pod information in <code>NodePublish</code>.</li>
<li>Admin installs CSI driver with CSI Driver Registrar <code>--requires-publish-pod</code>, e.g. by applying helm template from CSI diver author.</li>
<li>On a node, CSI Driver Registrar places UNIX domain socket to <code>/var/lib/kubelet/plugins/foo-reg.sock</code>.</li>
<li>Kubelet finds the socket there and calls <code>GetInfo</code> on it (no change from current behavior).</li>
<li>CSI Driver Registrar receives <code>GetInfo</code> call and passes CSI driver socket to kubelet (no change from current behavior).</li>
<li>Kubelet notifies CSI volume plugin that there is a new CSI driver installed and passes <strong>both</strong> CSI DriverRegistrar socket and CSI driver socket (no change from current behavior).</li>
<li>CSI volume plugin calls <code>GetCSIDriverInfo</code> on the CSI Driver Registrar socket.</li>
<li>CSI Driver Registrar, based on <code>--requires-publish-pod</code> presence on command line, returns <code>CSIDriverInfo</code> with <code>requires_publish_pod = true</code>.</li>
<li>CSI volume plugin stores this flag for the CSI driver.</li>
</ol>
<p>When a volume is being mounted for a pod:</p>
<ol>
<li>In CSI volume plugin <code>SetUpAt()</code>, the plugin checks if the CSI driver requires pod information and passes it as <code>volume_attributes</code> defined above.</li>
</ol>
<h3 id="csi-driver-uses-old-csi-driver-registrar-image">CSI driver uses old CSI Driver Registrar image</h3>
<p>When a CSI driver uses CSI Driver Registrar that does not implement <code>CSIRegistration</code> service.</p>
<ol>
<li>Admin installs CSI driver with old CSI Driver Registrar.</li>
<li>On a node, CSI Driver Registrar places UNIX domain socket to <code>/var/lib/kubelet/plugins/foo-reg.sock</code>.</li>
<li>Kubelet finds the socket there and calls <code>GetInfo</code> on it (no change from current behavior).</li>
<li>CSI Driver Registrar receives <code>GetInfo</code> call and passes CSI driver socket to kubelet (no change from current behavior).</li>
<li>Kubelet notifies CSI volume plugin that there is a new CSI driver installed and passes <strong>both</strong> CSI DriverRegistrar socket and CSI driver socket (no change from current behavior).</li>
<li>CSI volume plugin calls <code>GetCSIDriverInfo</code> on the CSI Driver Registrar socket.</li>
<li>CSI Driver Registrar does not implement <code>GetCSIDriverInfo</code> and returns an error.</li>
<li>CSI volume plugin marks the corresponding CSI driver as it does not need pod information.</li>
</ol>
<h2 id="alternatives">Alternatives</h2>
<ul>
<li>Add a new flag to <a href="https://github.com/kubernetes/kubernetes/blob/10688257e63e4d778c499ba30cddbc8c6219abe9/pkg/kubelet/apis/pluginregistration/v1alpha1/api.proto#L17"><code>PluginInfo</code> message</a>. This way, we would not need to implement new gRPC service. On the other way, <code>PluginInfo</code> is shared among all kubelet plugins (i.e. device plugins and CSI drivers) and such flag would pollute the API for device plugins.</li>
<li>Add flag to CSI itself. It was refused in CSI community, they don’t want to pass Kubernetes flags.</li>
</ul>
<h2 id="future-enhancements">Future enhancements</h2>
<p>In future we may add new flags / options to <code>CSIDriverInfo</code> message as Kubernetes CSI volume plugin may require more configuration how to work with each registered CSI driver.</p>

