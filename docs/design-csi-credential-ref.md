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
ServiceAccount. The deployment must grant atelet a narrowly scoped Role and
RoleBinding for each referenced Secret; this branch does not add a cluster-wide
Secret permission to the chart.

The node staging and publishing path is implemented and tested first because it
is the path currently blocking the JuiceFS POC. Controller-side CSI secret
propagation for CreateVolume and DeleteVolume is a separate follow-up and is
not part of this first validation.

For the POC, the required permission is equivalent to:

```yaml
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["juicefs-secret"]
  verbs: ["get"]
```

The Role must be created in the Secret namespace and bound to the deployed
atelet ServiceAccount. Do not apply this permission until the target namespace,
Secret name, and atelet ServiceAccount have been checked against live cluster
state.
