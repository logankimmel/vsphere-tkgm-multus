# Using Multus with 2NIC worker nodes on TKGm/vSphere

## Pre-reqs
* TKGm management cluster is created (tested with 2.5)
* vSphere with multiple networks (this example uses the networks: `user-workload` and `newport` )
* [ytt installed](https://carvel.dev/ytt/docs/v0.48.0/install/)
* kubectl installed and set to management cluster context
* Tanzu CLI

## Create Custom Cluster Class
This process involves creating a custom cluster class that allows for deploying worker nodes in a workload cluster that have multiple network interfaces. Creating custom clusterclasses is roughly documented [here](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.4/using-tkg/workload-clusters-cclass.html).

1. In Tanzu Kubernetes Grid 2.3.0 and later, after you deploy a management cluster, you can find the default ClusterClass manifest in the ~/.config/tanzu/tkg/clusterclassconfigs folder.

```bash
cp ~/.config/tanzu/tkg/clusterclassconfigs/tkg-vsphere-default-v1.2.0.yaml .
```

2. To customize your ClusterClass manifest, you create ytt overlay files alongside the manifest. Create a `custom` folder with the following files (example included in this repo)
```
custom
└── overlays
    ├── filter.yaml
    └── nics.yaml

```
Take note of the `nics.yaml` file that defines the network device patch I'm applying to the `VSphereMachineTemplate` resource

Use the default ClusterClass manifest to generate the custom clusterClass
```bash
ytt -f tkg-vsphere-default-v1.2.0.yaml -f custom/ > custom_cc.yaml
```

Install the custom clusterClass in the Management cluster:
```bash
kubectl apply -f custom_cc.yaml
```

You should see the following output when you run `kubectl get clusterclasses`:
```
NAME                                  AGE
tkg-vsphere-default-v1.1.1            21h
tkg-vsphere-default-v1.1.1-extended   20h
```

## Deploy new workload cluster that uses the custom clusterclass

1. Locate your configuration file you used for creating the management cluster (these are stored in: `~/.config/tanzu/tkg/clusterconfigs`). More info on the config file can be found [here](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.4/using-tkg/workload-clusters-deploy.html#prerequisites-0)

2. Copy the config file to your working director:
```bash
cp ~/.config/tanzu/tkg/clusterconfigs\{config_file}.yaml ./workload-1.yaml
```

3. Generate the custom workload cluster manifest
```bash
tanzu cluster create --file workload-1.yaml --dry-run > default_cluster.yaml
```
Using the overlay example in this directory, create the custom manifest:
```bash
ytt -f default_cluster.yaml -f cluster_overlay.yaml > custom_cluster.yaml
```

4. Deploy
```bash
tanzu cluster create -f custom_cluster.yaml
```

## Install Multus using Tanzu package manager

Follow the instructions outlined [here](https://docs.vmware.com/en/VMware-Tanzu-Packages/2023.11.21/tanzu-packages/packages-multus-mc.html)

In my case, the `newport` network interface was assigned to `eth0` on the worker node, so I used the following `NetworkAttachmentDefinition` for creating a macvlan attached to that device:
```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: vcenter-vm-second-nic
  namespace: default
spec:
  config: '{
    "cniVersion": "0.3.1",
    "plugins": [
    {
    "type": "macvlan",
    "master": "eth0",
    "mode": "bridge",
    "ipam": {}
    }
    ]
    }'
```

More advanced configurations can be found [here](https://www.cni.dev/plugins/current/)
