# Openshift Virtualization using Multus

## What we want to implement

Give direct access from a KubeVirt VM to an external VLAN using the secondary interface on worker nodes 

## Example of multus integration on Openshift Virtualization

 * Specific existing subnet
   * 10.6.85.0/24 =  vlan 1085
  
 * We use a **NodeNetworkConfigurationPolicy** object to add a tagged vlan from the trunked secondary interfaces in the cluster nodes
   
   * c7000 worker nodes with a secondary interface eno49, adding vlan 1085
   
                  apiVersion: nmstate.io/v1
                  kind: NodeNetworkConfigurationPolicy
                  metadata:
                    name: vlan-1085-policy-c7000
                  spec:
                    desiredState:
                      interfaces:
                        - name: eno49
                          state: up
                          type: ethernet
                        - description: VLAN using eno49
                          name: eno49.1085
                          state: up
                          type: vlan
                          vlan:
                            base-iface: eno49
                            id: 1085
                        - bridge:
                            options:
                              stp:
                                enabled: false
                            port:
                              - name: eno49.1085
                          description: Linux bridge with eno49.1085 as a port
                          ipv4:
                            enabled: false
                          name: bridge-1085
                          state: up
                          type: linux-bridge
                    nodeSelector:
                      hardware: c7000
   
How we bring multus connexion to your choosen namespaces

 * A **NetworkAttachementDefinition** is created to enable inserting a multus connexion on a KubeVirt VM (using a cnv-bridge)


                apiVersion: k8s.cni.cncf.io/v1
                kind: NetworkAttachmentDefinition
                metadata:
                  name: nad-1085
                  namespace: @namespace@
                spec:
                  config: >-
                    {
                      "name":"nad-1085",
                      "type":"cnv-bridge",
                      "cniVersion":"0.3.1",
                      "bridge":"bridge-1085",
                      "macspoofchk":true,
                      "ipam":{}
                    }     

 * Each VM needs to specify network connectivity and in this example to our multus vlan

              apiVersion: kubevirt.io/v1
              kind: VirtualMachine
              metadata:
                name: @name@
                namespace: @namespace@
                ...
              spec:
                ...
                   networks:
                    - multus:
                        networkName: nad-1085
                      name: nic-pxe1

