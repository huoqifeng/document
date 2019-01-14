# runC vs runQ


## runC

参考：

 - https://github.com/opencontainers/runc
 
## runQ

runq is a hypervisor-based Docker runtime based on runc to run regular Docker images in a lightweight KVM/Qemu virtual machine. The focus is on solving real problems, not on number of features.

Key differences to other hypervisor-based runtimes:

 - minimalistic design, small code base
 - no modification to existing Docker tools (dockerd, containerd, runc...)
 - coexistence of runq containers and regular runc containers
no extra state outside of Docker (no libvirt, no changes to /var/run/...)
simple init daemon, no systemd, no busybox
no custom guest kernel or custom qemu needed
runs on x86_64 and s390x
runc vs. runq
       runc container                   runq container
       +-------------------------+      +-------------------------+
       |                         |      | +---------------------+ |
       |                         |      | |                  VM | |
       |                         |      | |                     | |
       |                         |      | |                     | |
       |       application       |      | |     application     | |
       |                         |      | |                     | |
       |                         |      | |                     | |
       |                         |      | +---------------------+ |
       |                         |      | |     guest kernel    | |
       |                         |      | +---------------------+ |
       |                         |      |           qemu          |
       +-------------------------+      +-------------------------+
 ----------------------------------------------------------------------
                                host kernel

参考：

 - https://github.com/gotoz/runq
 

