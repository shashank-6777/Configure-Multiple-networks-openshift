# Configuring Multiple Networks in OpenShift v4.12.9

This guide provides step-by-step instructions for configuring multiple networks in OpenShift v4.12.9 using SR-IOV and macvlan network types. These network configurations can help you optimize network performance and segregation for your OpenShift workloads.

## Prerequisites

- OpenShift v4.12.9 cluster up and running.
- Nodes that support SR-IOV and macvlan network types.
- Admin access to the OpenShift cluster.

## SR-IOV Configuration

1. Identify SR-IOV Capable NICs:
   - SSH into each node.
   - Run the following command to identify SR-IOV capable NICs:
     ```bash
     lspci | grep Virtual Function
     ```
   - Note down the NIC names that support SR-IOV.

2. Configure SR-IOV in OpenShift:
   - Log in to the OpenShift cluster as an admin.
   - Create a new SR-IOV network YAML file (e.g., sriov-network.yaml) and define the SR-IOV network:

   - Apply the Network file:
     ```bash
     oc apply -f https://github.com/shashank-6777/Configure-Multiple-networks-openshift/blob/main/sriov-network.yaml
     ```
3. Validation for step 2:
     ```bash
     [root@vm_caasmng001 supervisor]# oc get sriovnetwork sriovnetwork -n openshift-sriov-network-operator -o yaml

    apiVersion: sriovnetwork.openshift.io/v1
    kind: SriovNetwork
    metadata:
      annotations:
        operator.sriovnetwork.openshift.io/last-network-namespace: sriov-app
      creationTimestamp: "2023-08-22T16:08:31Z"
      finalizers:
      - netattdef.finalizers.sriovnetwork.openshift.io
      generation: 2
      name: sriovnetwork
      namespace: openshift-sriov-network-operator
      resourceVersion: "9817135"
      uid: a77fbbd4-0b98-4ab3-8209-1ac66dd350b5
    spec:
      ipam: '{  "type": "host-local",  "subnet": "192.168.10.0/24",  "rangeStart": "192.168.10.171",  "rangeEnd":
    "192.168.10.181",  "routes": [{    "dst": "0.0.0.0/0"  }],  "gateway": "192.168.10.1"}'
      linkState: enable
      networkNamespace: sriov-app
      resourceName: netdevice
      spoofChk: "on"
      trust: "off"
     [root@vm_caasmng001 supervisor]#
    ```

4. Attach SR-IOV Network to a Pod:
   - Update your pod's YAML file to include the network attachment:
     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: sriov-app-1
       labels:
         app: nginx
       namespace: sriov-app
       annotations:
         k8s.v1.cni.cncf.io/networks: sriovnetwork
     spec:
       securityContext:
         allowPrivilegeEscalation: true
         privileged: true
       containers:
         - name: sriov-app-1
           image: 'registry.necdtvran.localdomain:5000/nec-nfvi/nfvi-ocp/4.12/centos/tools:latest'
           ports:
             - containerPort: 8080
               protocol: TCP
           command:
           - /sbin/init
           resources:
             limits:
               cpu: "8"
               openshift.io/netdevice: "1"
           requests:
               cpu: "8"
               openshift.io/netdevice: "1"
     ```
   - Apply the updated pod configuration:
     ```bash
     oc apply -f https://github.com/shashank-6777/Configure-Multiple-networks-openshift/blob/main/sriov-app-1.yaml
     ```
5. Validation for step 4:
    ```bash
    [root@vm_caasmng001 supervisor]# oc exec -it sriov-app-1 -n sriov-app -- bash
    [root@sriov-app-1 /]#
    [root@sriov-app-1 /]# ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
    2: eth0@if229: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP group default
    link/ether 0a:58:0a:80:02:24 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.128.2.36/23 brd 10.128.3.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::858:aff:fe80:224/64 scope link
       valid_lft forever preferred_lft forever
    79: net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether f6:8b:06:50:b7:fb brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.177/24 brd 192.168.10.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fe80::f48b:6ff:fe50:b7fb/64 scope link
       valid_lft forever preferred_lft forever
    [root@sriov-app-1 /]#
     ```
## macvlan Configuration

1. Identify Available Network Interfaces:
   - SSH into each node.
   - Run the following command to identify available network interfaces in my case its ens1f0:
     ```bash
     ip link show
     ```
   - Note down the interface names that you want to use for macvlan.

2. Create macvlan Network in OpenShift:
   - Log in to the OpenShift cluster as an admin.
   - Create a new NetworkAttachmentDefinition YAML file (e.g., macvlan-net.yaml) and define the macvlan network:
     ```yaml
     apiVersion: "k8s.cni.cncf.io/v1"
     kind: NetworkAttachmentDefinition
     metadata:
       name: macvlan-conf
     spec:
        config: |
         {
             "cniVersion": "0.3.1",
             "type": "macvlan",
             "master": "ens1f0",
             "mode": "bridge",
             "ipam": {
                 "type": "static",
                   "addresses": [
                    {
                      "address": "192.168.11.3/24"
                    }
                   ],
                   "routes": [
                      { "dst": "0.0.0.0/0", "gw": "192.168.11.254" }
               ]
             }
         }      
     ```

   - Apply the NetworkAttachmentDefinition:
     ```bash
     oc apply -f https://github.com/shashank-6777/Configure-Multiple-networks-openshift/blob/main/macvlan-net.yaml
     ```

3. Validation for step 2:
     ```yaml
     [root@vm_caasmng001 supervisor]# oc get net-attach-def macvlan-conf -o yaml
     apiVersion: k8s.cni.cncf.io/v1
     kind: NetworkAttachmentDefinition
     metadata:
       creationTimestamp: "2023-08-25T11:32:31Z"
       generation: 1
       name: macvlan-conf
       namespace: macvlan-app
       resourceVersion: "12254752"
       uid: 4e6d0e10-d3aa-4a3a-bb16-31d7ef6740a7
     spec:
       config: |
         {
            "cniVersion": "0.3.1",
            "type": "macvlan",
            "master": "ens1f0",
            "mode": "bridge",
            "ipam": {
                "type": "static",
                  "addresses": [
                   {
                     "address": "192.168.11.3/24"
                   }
                  ],
                  "routes": [
                     { "dst": "0.0.0.0/0", "gw": "192.168.11.254" }
                ]
            }    
         }
     ```
4. Attach macvlan Network to a Pod:
   - Update your pod's YAML file to include the network attachment:
     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: support-app-1
       labels:
         app: support-app-1
       namespace: macvlan-app
       annotations:
         k8s.v1.cni.cncf.io/networks: "macvlan-conf"
     spec:
       containers:
         - name: support-app-1
           image: 'registry.necdtvran.localdomain:5000/nec-nfvi/nfvi-ocp/4.12/support-tools'
           imagePullPolicy: Never
           ports:
             - containerPort: 8080
           command: ["sleep", "30d"]
           nodeSelector:
              kubernetes.io/hostname: worker03.dtcl1.necdtvran.localdomain

     ```
   - Apply the updated pod configuration:
     ```bash
     oc apply -f https://github.com/shashank-6777/Configure-Multiple-networks-openshift/blob/main/macvlan-app-1.yaml
     ```
5. Validation for step 4:
     ```yaml
     [root@vm_caasmng001 supervisor]# oc exec -it support-app-1 -- bash
     [root@support-app-1 /]# ip a
     1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
     2: eth0@if176: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc noqueue state UP group default
    link/ether 0a:58:0a:80:02:fe brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.128.2.254/23 brd 10.128.3.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::858:aff:fe80:2fe/64 scope link
       valid_lft forever preferred_lft forever
     3: net1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue state UP group default
    link/ether c6:79:cf:8d:e9:47 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.11.3/24 brd 192.168.11.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fe80::c479:cfff:fe8d:e947/64 scope link
       valid_lft forever preferred_lft forever
     [root@support-app-1 /]#

     ```

## Conclusion

By following this guide, you have successfully configured multiple networks using SR-IOV and macvlan in OpenShift v4.12.9. These network configurations can help optimize network performance and segregation for your OpenShift workloads.

Feel free to explore further options and fine-tune your network configurations based on your specific requirements.
