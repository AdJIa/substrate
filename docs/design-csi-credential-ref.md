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

## POC result

Validated on the kind `kagent-substrate` cluster on 2026-11-18 with the fork
commit `5ba30577a`:

- `ate-api-server` and `atelet` ran the forked images.
- A new ActorTemplate containing `credentialRef.namespace=juicefs-poc` and
  `credentialRef.name=juicefs-secret` was accepted by the forked API and client.
- A new Actor carried the reference through the atelet workload request.
- atelet mounted the JuiceFS volume at the Actor path.
- The JuiceFS NodePublish log reported the six expected Secret keys and the
  mount completed in about 594 ms.
- Actor status retained the existing CSI volume context and did not contain
  Secret data. Logs did not contain Secret values.

The complete Actor Ready check remained blocked after the successful mount by a
separate gVisor v6 asset fetch failure. The asset URL at `172.80.0.1:8000`
returned `InvalidArgument`, so the Harness wakeup probe never became ready.
This is an environment asset-service failure; it is not reported as a passing
Agent E2E test for this change.
