## Installing Fenrir
At this current stage fenrir works on a Single Node, and the following are the instructions on how to install it and optimize it's streaming.  

## Prerequisite configurations  
All the resource manifests are at `examples/installation`  
You will need to navigate to the installation directory to edit a few manifests before installation.  

### Kubernetes Version  
Ensure your cluster version is either:  
* v1.34 -> v1.35: with the following feature gates enabled:
    * DynamicResourceAllocation
    * DRAConsumableCapacity
* v1.36: the previous feature gates are generally available  

Before installing fenrir, you'll need to configure 3 things:    
* deviceclass
* wolf-agent
* applications

### Wolf Device Class

The DeviceClass contains the default lobby values, the relevant of which are the renderNodes,  
by default wolf wolf uses `/dev/dri/renderD128` to create he wayland sockets, however in multi-gpu setups, and DRA mounts, your Pod may only have `/dev/dri/renderD129` present, which may cause wolf to crash.  
You'll need to edit `03-deviceclass.yaml` before installation to have the correct render node.  
```
wayland_render_node: "/dev/dri/renderD12X"
runner_render_node: "/dev/dri/renderD12X"
```

### Wolf-Agent GPU Mount / Resource Claim

The wolf-agent requires a GPU to render / function.  
Since we're most likely installing in kubernetes v1.35+, we'll be using resource claims.  

The node where there wolf-agent will be installed will also be hosting the applications for the users, so keep that in mind.  

The current setup uses AMD GPU Operator, in case of nvidia, you'll need to modify it.  
You will need to edit `01-wolf-agent-standalone.yaml` at:  
* **ResourceClaim**: `shared-amd-gpu`  
* **Deployment**: `resourceClaims` & wolf container's `resources.claims`  

### Applications

Once you've updated the wolf-agent's resourceClaims, you'll need to do the same to which ever application you want to launch.  
Go to `.spec.template.spec.resourceClaims` in each application and update it:  
* same resource claim you used on the wolf-agent  
* another resource claim available on the same node as the wolf-agent  

This is to ensure that the statefulsets / application lobbies are scheduled on the same node as the wolf-agent, since the direwolf-operator injects a resource claim that requests a lobby from the wolf-agent.  
Basically: the kube-scheduler needs both the lobby and gpu resource claims to target the same node in order for it to successfully schedule the pod.  


## Installation
Currently we're still under development and testing, so we're creating the namespace called `direwolf-dev` with the needed privileges to create a hostPath mount for the wolf-agent.  

The installation is straight forward, apply all the manifests at `examples/installation`:  
`kubectl apply -f examples/installation`  

This will create:
* Namespace: direwolf-dev
* ResourceClaim: **You created it in the previous step**
* ServiceAccounts:
    * wolf-dra
    * direwolf-operator
    * direwolf-moonlight-proxy
* ClusterRoles:
    * wolf-dra
    * direwolf-operator
    * direwolf-moonlight-proxy
* ClusterRoleBinding:
    * wolf-dra
    * direwolf-operator
    * direwolf-moonlight-proxy
* Service: **Both services should share the same ip address**
    * wolf-agent
    * direwolf
* Deployments:
    * wolf-agent-dev:
        * wolf-agent: The DRA agent responsible for mounting wolf's wayland + pulseaudio sockets 
        * wolf: Responsible for creating the needed socket and streaming to the user
    * direwolf-operator-dev
    * direwolf-moonlight-proxy-dev

Once installation is complete, you'll need to ensure that the `moonlight-proxy` and `wolf-agent` are sharing the same IP:
* moonlight-proxy: to pair with users, authenticate and inform the clients of the available games
* wolf-agent: to stream the applications that are started
Run the following to get their ip addresses:  
`kubectl get svc -n direwolf-dev`  
**If they don't share the same ip, streaming will fail.**  

Once all deployments are ready in the `direwolf-dev` namespace you will need to pair your moonlight clients.  

## Pairing your devices with fenrir
We don't have a GUI yet, so you'll need to get the logged pairing URL from the moonlight-proxy.  
Start by attaching to the moonlight-proxy logs:  
`kubectl logs -n direwolf-dev deployments/moonlight-proxy -f`  

Then use the this command to get the ip address:  
`kubectl get svc -n direwolf-dev`  

Use the direwolf ip to in your moonlight client, and once the pairing process starts, you'll find the pairing URL in the moonlight-proxy logs.  
Copy it to your browser of choice, set a username / device identifier before writing the pairing code.  

Once the devices are paired successfully, you can check your pairing information by running:  
`kubectl get pairings -n direwolf-dev`  
You'll need the pairing name to add your device/s to the default profile in `examples/profile.yaml`.   

once you've added your pairing name, apply the `profile.yaml` manifest.  
`kubectl apply -f profile.yaml`  

## Applications
I have not figured out validation yet, so a lot of failures are silent, which is annoying to troubleshoot.  

However, this is a quick and simple introduction to the application CRD.  
```
apiVersion: direwolf.games-on-whales.github.io/v1alpha1
kind: App
metadata:
  name: firefox
  namespace: direwolf-dev
spec:
  appAssetWebP: <base64 of a webp image, will allow for config map references soon>
  id: <id needs to be unique, otherwise it'll cause issues>
  isHDRSupported: <I'm not sure if wolf supports HDR yet, but we're keeping this for when I know for certain>
  title: <the title shown to the moonlight-client>
  deviceClassName: <in case you have different device class, i'm not sure what to do with this yet>
  volumeClaimTemplates:
    <same as a statefulset volume claim template>
    <You can add as many pvc as you'd like>
  template:
  <whatever under template is a pod spec>
  <You can place anything a normal pod has>
  <for example, volume mounts pointing to a PVC / NFS storage that has games / executables related to the application>
```

you can apply the application, and in a few seconds it should appear in your moonlight client if you've already applied the `profile.yaml` containing your pairing name.  

next, you launch it from the moonlight-client and a statefulset should appear in the `direwolf-dev` namespace.  
If that doesn't happen, inspect the direwolf-operator logs.  
It usually means that the App CRD and or Statefulset failed verification by the kubernetes api.  
`kubectl logs -n direwolf-dev deployments/direwolf-operator-dev -f`  
The logs usually point to what failed validation, but I have to admit, I need to implement better logging / crd validation.  

once the application manifest is correctly configured, it should create a statefulset pod on the same node as the wolf-agent.  
Once the pod is in the "Ready" state, the stream can start / resume.  
Sometime wolf crashes, so when that happens you'll need to restart the wolf-agent pod.  

## Post-Installation
Some considartions for fine-tuning the wolf stream.  
Streaming requires the least number of hops.  
so, if possible ensure that your wolf-agent host / node is the one responsible for the l2 lease.  

#### Cilium
you can check the owner of the loadbalancer lease through:  
`kubectl get lease -n kube-system`  
you can use `ciliuml2announcementpolicies.cilium.io` to control which node has the l2 lease on the loadbalancer service.  

#### Metallb
you can check the owner of the loadbalancer lease through:  
`kubectl get servicel2statuses.metallb.io -A`  
you can use `l2advertisements.metallb.io` to control which node has the l2 lease on the loadbalancer service.  

if it's the same node as the wolf-agent, then there is a minimal number of hops between your stream and the wolf-agent.  
otherwise, kubernetes may route your stream through multiple nodes before reaching your moonlight-client.  

worst case scenario it makes the stream unustable.  