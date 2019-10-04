# ceos-docs
cEOSr is a containerized version of EOS, design to run as the network control plane in a Kubernetes cluster.  It can be deployed in parallel with other CNI providers, allowing different network security policy implementations to be used with the EOS network control plane.  In this release, cEOSr has been tested with the Calico CNI for policy, and is designed as a layer 3 solution, creating a BGP Peering relationship to the top of rack (ToR) switch.  
## Design Goals
There are a few guiding design goals with cEOSr:
* Provide a Kubernetes native deployment model
* Allow for easy deployment and redeployment of cluster nodes
* Provide an operationally consistent networking model for administrators
* Provide the benefits of EOS from a management, observability, feature, and supportability in a Kubernetes networking environment.

cEOSr achieves these goals by the following:

* Using a daemonset to deploy ceos containers on each node in the cluster
* Providing configuration options directly in the deployment YAML file
* Allowing standard EOS configuration 
* Using the same binary as any other EOS based device
* Support for streaming telemetry and management through Arista CloudVision

## Supported Annotations for cEOS

|Parameter | Description | Example |Optional|
|----------|-------------|---------|--------|
|arista/bgp-peer-ip-1|IP address of BGP Peer | arista/bgp-peer-ip-1="172.20.3.1"|no|
|arista/bgp-peer-ip-2|IP address of second BGP Peer for redundant links| arista/bgp-peer-ip-2="172.20.4.1"|yes|
|arista/bgp-local-as |ASN for the Kubernetes nodes.  If using iBGP this should match the ToR| arista/bgp-local-as="65003"|no|
|arista/bgp-remote-as-1 |ASN for the BGP Peer if not using iBGP| arista/bgp-remote-as-1="65001"|yes|
|arista/bgp-remote-as-2 |ASN for the second BGP Peer if not using iBGP| arista/bgp-remote-as-2="65002"|yes|
|LOOPBACK_INTERFACE| Interface number for loopback interface creation | LOOPBACK_INTERFACE=1 |yes|
|LOOPBACK_IP|IP address of the loopback interface with mask|LOOPBACK_IP=192.16.1.1/32|yes|
|LOOPBACK_DESCRIPTION|Description of the loopback interface| LOOPBACK_DESCRIPTION="node 1 loopback"|yes|


## ConfigMap support for cEOS

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

## Deployment Guide
For a more in-depth deployment guide see : [DEPLOY.md](DEPLOY.md)
## Example Configuration
For an example configuration see : [EXAMPLE.md](EXAMPLE.md)
