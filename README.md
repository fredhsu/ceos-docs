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

