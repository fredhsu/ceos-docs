
# Step 1 - install Calico policy only CNI: 

https://docs.projectcalico.org/v3.4/getting-started/kubernetes/installation/other

Download `calico.yaml`:


    curl \
    https://docs.projectcalico.org/v3.4/getting-started/kubernetes/installation/hosted/kubernetes-datastore/policy-only/1.7/calico.yaml \
    -O

Edit the following field in `calico.yaml` to match your CIDR for the cluster : 

            - name: CALICO_IPV4POOL_CIDR
              value: "10.153.0.0/16"

Apply manifest

    kubectl apply -f calico.yaml

# Step 2 - Install ceos

Download `ceosr-init-0.7.yaml`:

    curl https://storage.googleapis.com/ceos/ceosr-init-0.7.yaml -O

Edit the following fields inside `ceosr-init-0.7.yaml` to insert your CV IP address.  Please be careful of indentation when dealing with YAML:

    - name: "CLOUDVISION_IP"
      value: "10.90.224.175"

Annotate the Kubernetes nodes with the BGP configuration relevant for the node:

    kubectl annotate node <k8snode> BGP_PEER=172.20.14.1 BGP_ASN=65130

(optional) There are a number of optional parameters you can also specify via annotations:

|Parameter | Description | Example |
|----------|-------------|---------|
|LOOPBACK_INTERFACE| Interface number for loopback interface creation | LOOPBACK_INTERFACE=1 |
|LOOPBACK_IP|IP address of the loopback interface with mask|LOOPBACK_IP=192.16.1.1/32|
|LOOPBACK_DESCRIPTION|Description of the loopback interface| LOOPBACK_DESCRIPTION="node 1 loopback"|
|arista/bgp-peer-ip-2|IP address of second BGP Peer for redundant links| arista/bgp-peer-ip-1="172.20.4.1"|
|arista/bgp-remote-as-1 |ASN for the BGP Peer if not using iBGP| arista/bgp-remote-as-1="65001"|
|arista/bgp-remote-as-2 |ASN for the second BGP Peer if not using iBGP| arista/bgp-remote-as-2="65002"|
|arista/bgp-peer-ip-2|IP address of second BGP Peer for redundant links| arista/bgp-peer-ip-1="172.20.4.1"|


(optional) If you need to apply a chunk of static config to every pod you can also create a configmap in Kubernetes that will store and configure all cEOS pods with generic configuration.  NOTE: These commands must be fully expanded (i.e. `interface Ethernet 1`, not `int eth 1`), and no syntax checking will occur.  If there are syntax errors with the commands it may prevent the pod from starting.  In this example I'm adding a few lines of static config for SSH to be remapped to port 8022 and adding an NTP server.  This config will be placed on every cEOS pod.

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: ceos-configmap
      namespace: default
    data:
      ceos-config-data: |
        ntp-server 172.22.22.50
        management ssh
        server-port 8022

You will then need to add this configmap to your YAML specification for cEOS, namely as a volumeMount under ceos-init and a volume under the pod.  These are noted in the following snippets with a comment:

Entry for the ceos-init container:

      initContainers:
      - image: fredhsu/ceos-init:0.7
        name: ceos-init
        command: ["/ceos-init", "-cni=false"]
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: NODE_AS
            valueFrom:
              fieldRef:
                fieldPath: metadata.labels['asn']
          - name: "BGP_AS"
            value: "65130"
          - name: "BGP_PEER"
            value: "172.20.14.1"
          - name: "CLOUDVISION_IP"
            value: "10.90.224.175"
        volumeMounts:
        - name: kickstart-config
          mountPath: /config/
        # Add the following:
        - name: configmap-config
          mountPath: /config/ceos-configmap

Entry for the volumes section:

    volumes:
    - name: kickstart-config
      hostPath:
        path: /opt/cni/ceos
    # Add the following:
    - name: configmap-config
      configMap:
        name: ceos-configmap



For the current test we are running ceos in the default namespace, to run it on the master you will need to remove the taint if you have one:
`kubectl taint nodes --all node-role.kubernetes.io/master-`

Apply manifest:

    kubectl apply -f ceosr-init-0.7.yaml

Currently ceos takes about 2 mins to boot, give it a bit of time before checking on it.  If all goes well you should see routes show up in the node's routing table, and you can `kubectl exec` into the pod to run commands.  For example:
`kubectl exec -it <ceos podname> -- FastCli -p 15 -c "show version"`


# Configuring the ToR
It is convienent to use bgp dynamic peers for the ToR configuration so that new nodes can be dynamically added to the network without needing to change the switch config.  Since we are using iBGP here, you will need to configure the ToR as a route reflector:

    router bgp 65003
      bgp listen range 172.20.3.0/24 peer-group kubedemo1 remote-as 65003
      neighbor kubedemo1 peer-group
      neighbor kubedemo1 route-reflector-client
      neighbor kubedemo1 maximum-routes 12000

