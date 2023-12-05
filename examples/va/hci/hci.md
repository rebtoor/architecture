# Configuring networking and deploy the OpenStack control plane

## Assumptions

- A storage class called `local-storage` should already exist.

## Initialize

Switch to "openstack" namespace
```
oc project openstack
```
Change to the hci directory
```
cd architecture/examples/va/hci
```
Edit the [values.yaml](values.yaml) file to suit your environment.
```
vi values.yaml
```
Alternatively use your own copy of `values.yaml` and edit 
[kustomization.yaml](kustomization.yaml) to use a that copy.
```
resources:
  - values-ci-framework.yaml
```

Generate the control-plane and networking CRs.
```
kustomize build > control-plane.yaml
```

## Workarounds

The move to kustmoize is not complete so we need these for now.

### Interface Name
Identify your interface:
```
oc debug node/master-0 -- chroot /host nmcli \
    -g GENERAL.DEVICES,IP4.ADDRESS con show 'Wired Connection' | \
    grep -B 1 168.122 |  grep enp
```
Ensure it is correctly referenced in `control-plane.yaml`. kustomize
is not yet replacing 100% of the instances from `values.yaml` but
should before PR39 merges.

### Node Names
Identify your nodes with `oc get nodes` and ensure the
`NodeNetworkConfigurationPolicy` name `nodeSelector` hostname are set
accordingly.
```
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  labels:
    osp/nncm-config-type: standard
  name: master-1
...
  nodeSelector:
    kubernetes.io/hostname: master-0
    node-role.kubernetes.io/worker: ""
```
### L2Advertisement Interface Names
The `L2Advertisement` interface name like
[enp6s0.20](https://github.com/openstack-k8s-operators/architecture/blob/main/va/hci/stage3/metallb_l2advertisement.yaml#L28)
and the `NodeNetworkConfigurationPolicy` like
[enp6s0.20](https://github.com/openstack-k8s-operators/architecture/blob/main/va/hci/stage3/ocp_node_0_nncp.yaml#L34)
should match and use a name like
`enp6s0.20` and not `internalapi`.

## Create CRs
```
oc apply -f control-plane.yaml
```

Wait for NNCPs to be available
```
oc wait nncp -l osp/nncm-config-type=standard --for condition=available --timeout=300s
```

### Todo
- Update `../../../lib/networking/netconfig.yaml` to use kustomize
- Update `../../../lib/networking/netattach_*` to use kustomize
- This is a place holder until the READMEs is moved to [docs](../../../docs/)


