# microk8s notes

- kubelet process
  - Located at /var/lib/kubelet
  - Since 1.21 kubelet was consolidated into daemon-kubelite process
  - Arguments of kubelet process can be found at /var/snap/microk8s/current/args/kubelet

- Node authorization mode
  - Authorization mode is set to "AlwaysAllow per default" (visible in /var/snap/microk8s/current/args/kube-apiserver)
  
- Admission plugins
  - visible in /var/snap/microk8s/current/args/admission-control-config-file.yaml

- containerd
  - Configuration with CR handlers available at /var/var/snap/microk8s/current/args/containerd-template.toml 

- kube-proxy
  - Configuration available at /var/snap/microk8s/current/args/kube-proxy
  - Per default uses iptaples mode -> Random load balancing of service endpoints
    kube-proxy
    - Per default uses iptaples mode -> Random load balancing of service endpoints
