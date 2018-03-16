Run Linux containers on Windows nodes
=====================================

This needs a short doc written up for feedback within SIG-Windows.

In short - we need to either:
1) Have a modification to kubelet and/or CRI to be able to detect the OS used by an image, and add extra flags if needed to run a Linux container on a Windows node
2) Run two kubelets with different values for `beta.kubernetes.io/os=` on the same node. The one for `beta.kubernetes.io/os=Linux` would somehow set the platform flag to Linux. That could even be done at the dockerd/containerd level since you can run multiple daemons side by side.


References:

https://github.com/moby/moby/pull/34859 



References
----------

* https://trello.com/c/QmOwGrQR/49-run-linux-containers-on-windows-nodes
* https://github.com/kubernetes/kubernetes/pull/58751