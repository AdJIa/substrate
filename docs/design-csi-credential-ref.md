# CSI credential references

This POC adds a Kubernetes Secret reference to an external volume template.
The reference is carried through the control-plane and atelet workload APIs,
but Secret data is resolved only by atelet immediately before CSI node mount.

## Data flow

```text
ActorTemplate.external.template.credentialRef
  -> workload spec external credentialRef
  -> atelet Kubernetes Secret GET
  -> CSI NodeStageVolumeRequest.Secrets
  -> CSI NodePublishVolumeRequest.Secrets
```

The Secret data is not stored in ActorStatus, snapshots, volumeContext, logs, or
the actor environment. Missing or forbidden Secrets fail the mount; atelet does
not retry with an empty map.

The first POC accepts a Kubernetes Secret reference with `namespace` and `name`
and passes every `data` entry to CSI using the entry name as the CSI Secret key.
The reference is namespaced and is authorized by Kubernetes RBAC on the atelet
ServiceAccount. A later change can add per-driver credential providers without
changing the external volume lifecycle.

This change intentionally covers the node staging and publishing path first,
which is the path currently blocking the JuiceFS POC. Controller-side CSI
secret propagation for provision and deletion is a separate follow-up.
