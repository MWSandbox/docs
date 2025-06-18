# microk8s notes

- kubelet process
  - Located at /var/lib/kubelet
  - Since 1.21 kubelet was consolidated into daemon-kubelite process
  - Arguments of kubelet process can be found at /snap/microk8s/current/default-args/kubelet

- Node authorization mode
  - Authorization mode is set to "AlwaysAllow per default" (visible in /snap/microk8s/current/default-args/kube-apiserver)
  
- Admission plugins
  - visible in /snap/microk8s/current/default-args/admission-control-config-file.yaml