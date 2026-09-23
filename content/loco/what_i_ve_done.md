---
title: What I've done
draft: false
type: "docs"
toc: true
---


## Bios

I\'ve done nothing to the bios, but there are things we could do:

- Disable core performance boost
- If applicable, determinism slider -\> performance determinism
  (Dell/HPE/Supermicro). Disables CPPC auto-boost.
- Set the NUMA nodes per socket to be one NUMA domain per CCD.

## Linux

### Disable the swap

``` bash
sudo swapoff -a
sudo vi /etc/fstab # comment the swap line
```

### Lock govenor performance

``` bash
sudo modprobe msr
echo msr | sudo tee /etc/modules-load.d/msr.conf
sudo cpupower frequency-set --governor performance
```

### Disable CPU boost

``` bash
echo 0 | sudo tee /sys/devices/system/cpu/cpufreq/boost
```

### Cstates

``` bash
# /etc/default/grub — append inside existing quotes:
GRUB_CMDLINE_LINUX="processor.max_cstate=1 nmi_watchdog=0"
sudo update-grub
sudo reboot
```

### Disable CPU Boost

``` bash
# Disable CPB/Turbo boost
echo 0 | sudo tee /sys/devices/system/cpu/cpufreq/boost
```

### Allow Transparent Huge Pages, when asked

``` bash
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

### Ensure host name is in `/etc/hosts`

``` bash
hostname -s                  # remember this value, call it $H
grep "$(hostname -s)" /etc/hosts || echo "127.0.0.1 $(hostname -s)" | sudo tee -a /etc/hosts
```

## Slurm configuration

### Munge (authentication)

``` bash
sudo apt install munge        # RHEL: dnf install munge munge-libs
# same key on every node — one node here, so just install:
sudo systemctl enable --now munge
systemctl status munge        # active (running)
```

Clock sync matters for munge; single machine = trivially fine. NTP still
recommended if you\'ll grow the cluster.

### Install `slurm` packages

``` bash
sudo apt install slurm-wlm slurmctld slurmd slurm-client
```

### Configuration files

`/etc/slurm/slurm.conf`:

``` conf
# /etc/slurm/slurm.conf
ClusterName=dantzig
SlurmctldHost=pricer

AuthType=auth/munge
CredType=cred/munge

SlurmUser=slurm
SlurmctldPort=6817
SlurmdPort=6818
SlurmctldTimeout=300
SlurmdTimeout=300

StateSaveLocation=/var/spool/slurmctld
SlurmdSpoolDir=/var/spool/slurmd
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log

ProctrackType=proctrack/cgroup
JobacctGatherType=jobacct_gather/cgroup

SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
TaskPlugin=task/cgroup,task/affinity

# Default value of memory per CPU
DefMemPerCPU=4096

# Node config
NodeName=pricer Sockets=2 CoresPerSocket=24 ThreadsPerCore=2 RealMemory=250000 State=UNKNOWN

# Partition config
PartitionName=debug Nodes=pricer Default=YES DefaultTime=01:00:00 MaxTime=02:00:00 State=UP
```

Notes:

- In the partition configuration, we can use AllowGroups to specify
  which accounts can access each partition.
- When configuring the `opt` and `brainiac` into the cluster, those will
  compose a new partition.
- Althought I\'ve not yet configured, but we can look into `preemption`,
  `checkpoints` and priorities.

`/etc/slurm/cgroup.conf`:

``` bash
# /etc/slurm/cgroup.conf
CgroupAutomount=yes
ConstrainCores=yes
ConstrainRAMSpace=yes
ConstrainSwapSpace=yes
AllowedSwapSpace=0
```

### Slurm folders

``` bash
sudo mkdir -p /var/spool/slurmctld /var/spool/slurmd /var/log/slurm
sudo chown slurm: /var/spool/slurmctld /var/spool/slurmd /var/log/slurm
```

### Start slurm

This should be done after munge is running.

``` bash
sudo systemctl enable --now slurmctld slurmd
systemctl status slurmctld slurmd
```

### Test the slurm

``` bash
sinfo                              # debug up, node idle
scontrol show node $H              # CPUs=96 AllocCPUs=0, State=IDLE
srun -N1 hostname                  # returns $H
squeue                             # empty queue, no errors
```

## ssh

Changed ssh to only allow for key-based connection, not passwords.

In `/etc/ssh/sshd_config`:

    PubkeyAuthentication yes
    PasswordAuthentication no
    KbdInteractiveAuthentication no
    PermitRootLogin no

## Apptainer

``` bash
sudo apt install apptainer
sudo apt install squashfuse fuse2fs gocryptfs
sudo apt install -y uidmap
```

### AppArmor allowlist for the apptainer starter

Ubuntu 23.10+/24.04 denial signature (rootless exec):

```bash
ERROR  : Could not write info to setgroups: Permission denied
ERROR  : Error while waiting event for user namespace mappings: no event received
```

Mechanism: with `kernel.apparmor_restrict_unprivileged_userns=1`,
AppArmor transitions any userns-creating process into the
`unprivileged_userns` profile, which denies `sys_admin` inside the
namespace (dmesg shows `DENIED ... capname="sys_admin"`). A profile with
an explicit `userns` rule is **exempt from that transition**. Scope it
to exactly the starter binary and only apptainer escapes; every other
unconfined process stays restricted.

1.  Check for an existing profile --- skip to step 3 if one is already
    loaded:

    ``` bash
    ls /etc/apparmor.d/ | grep -iE 'apptainer|singular'
    sudo aa-status | grep -iE 'apptainer|singularity'
    ```

2.  Write the profile:

    ``` bash
    sudo tee /etc/apparmor.d/apptainer << 'EOF'
    abi <abi/4.0>,
    include <tunables/global>
    profile apptainer /usr/lib/x86_64-linux-gnu/apptainer/bin/starter{,-suid} flags=(unconfined) {
      userns,
      include if exists <local/apptainer>
    }
    EOF
    ```

    `flags=(unconfined)` does not weaken anything: an unconfined-flagged
    profile\'s permissions are determined entirely by the profile
    itself, and this one grants exactly `userns`. Starter behavior is
    otherwise unchanged.

3.  Reload + verify:

    ``` bash
    sudo systemctl reload apparmor
    sudo aa-status | grep -i apptainer
    ```

4.  Test as an unprivileged user --- restriction must stay on:

    ``` bash
    apptainer exec ubuntu.sif cat /etc/os-release
    sysctl kernel.apparmor_restrict_unprivileged_userns    must still = 1
    sudo dmesg | tail -n 30 | grep -iE 'apparmor.*(DENIED|audit)'    no NEW denials
    ```

Rollback:
`sudo rm /etc/apparmor.d/apptainer && sudo systemctl reload apparmor`.
