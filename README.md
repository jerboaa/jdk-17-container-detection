# OpenJDK 17 Container Detection Demo

This repository contains resource files used for the OpenJDK 17 container
detection screencast illustrating cgroups v2 support on an Red Hat Enterprise
Linux 9 system with and Minikube installed.

## Demo Application Deployment

In order to re-recreate the undertow deployment use:

```
$ git clone https://github.com/jerboaa/jdk-17-container-detection
$ cd jdk-17-container-detection
$ kubectl create -f fedora-undertow-jdk17.yaml && \
  kubectl expose deployment fedora-undertow-jdk17 --type=LoadBalancer --port=8080 && \
  kubectl get service fedora-undertow-jdk17
```

Access the endpoint via the external IP address shows via the last `kubectl get service` command. For example:

```
$ curl -w '\n' http://10.109.124.25:8080
Hello World
```

## Pod Name Helpers

In order to get the pod name of the deployed application, one can use this trick:

```
$ kubectl get pods -l app=fedora-undertow-jdk17 -o custom-columns=POD:.metadata.name
POD:
-------------------------------------
fedora-undertow-jdk17-c667567cf-5c6jk
```

Using this, we can define an alias to get the pod name of the current deployment regardless how often we redeploy:

```
$ alias fedora-undertow-jdk17-pod='kubectl get pod -l app=fedora-undertow-jdk17 -o custom-columns=POD:.metadata.name | tail -n1'
$ fedora-undertow-jdk17-pod
fedora-undertow-jdk17-c667567cf-5c6jk
```

## Used JAVA_OPTS

The demo uses the following two options for a) container trace logging and b) gc info logging.

JVM Option | Description
-------------------------------------------------------------
`-Xlog:os+container=trace` | Turns on container trace level logging
-------------------------------------------------------------
`-Xlog:gc=info             | Turns on GC info level logging

## Describe Kubernetes Deployment

In order to see resource quotas on the Kubernetes level, describing the deployment can be useful:

```
$ kubectl describe deployment fedora-undertow-jdk17
```

## Inspecting Container Trace Logs

In order to see full logs of the deployed pod use (uses alias `fedora-undertow-jdk17-pod` defined above):

```
$ kubectl logs --tail=-1 $(fedora-undertow-jdk17-pod)
```

Restricting this to the first 10 lines of log output will show the relevant container trace output. Feel free to adjust the `-nXX` option to your needs:

```
$ kubectl logs --tail=-1 $(fedora-undertow-jdk17-pod) | head -n10
[0.020s][trace][os,container] OSContainer::init: Initializing Container Support
[0.021s][debug][os,container] Detected cgroups v2 unified hierarchy
[0.021s][trace][os,container] Path to /memory.max is /sys/fs/cgroup//memory.max
[0.021s][trace][os,container] Raw value for memory limit is: 2147483648
[0.021s][trace][os,container] Memory Limit is: 2147483648
[0.021s][info ][os,container] Memory Limit is: 2147483648
[0.021s][trace][os,container] Path to /cpu.max is /sys/fs/cgroup//cpu.max
[0.022s][trace][os,container] Raw value for CPU quota is: 300000
[0.022s][trace][os,container] CPU Quota is: 300000
[0.022s][trace][os,container] Path to /cpu.max is /sys/fs/cgroup//cpu.max
[...]
```

## Inspecting GC Info Logs

In order to see the GC algorithm in use on the deployed pod (uses alias `fedora-undertow-jdk17-pod` defined above):

```
$ kubectl logs --tail=-1 $(fedora-undertow-jdk17-pod) | grep Using
[0.007s][info ][gc          ] Using G1
```

## VM.info JCMD

A running JVM can be inspected for example via the VM.info JCMD. For example to see the full `VM.info` output use (note that the deployed application runs as PID `1`):

```
$ kubectl exec $(fedora-undertow-jdk17-pod) -- jcmd 1 VM.info
1:
#
# JRE version: OpenJDK Runtime Environment 21.9 (17.0.2+8) (build 17.0.2+8)
# Java VM: OpenJDK 64-Bit Server VM 21.9 (17.0.2+8, mixed mode, sharing, tiered, compressed oops, compressed class ptrs, g1 gc, linux-amd64)

---------------  S U M M A R Y ------------

Command Line: -Xlog:gc=info -Xlog:os+container=trace /deployments/undertow-servlet.jar

Host: QEMU Virtual CPU version 2.5+, 4 cores, 2G, Fedora release 35 (Thirty Five)
Time: Tue Feb 22 13:35:56 2022 UTC elapsed time: 64.876037 seconds (0d 0h 1m 4s)

---------------  P R O C E S S  ---------------

Heap address: 0x00000000e0000000, size: 512 MB, Compressed Oops mode: 32-bit

CDS archive(s) mapped at: [0x0000000800000000-0x0000000800be4000-0x0000000800be4000), size 12468224, SharedBaseAddress: 0x0000000800000000, ArchiveRelocationMode: 0.
Compressed class space mapped at: 0x0000000800c00000-0x0000000840c00000, reserved size: 1073741824
Narrow klass base: 0x0000000800000000, Narrow klass shift: 0, Narrow klass range: 0x100000000
[...]
```

In order to only see the cgroup information, use something like this:

```
$ kubectl exec $(fedora-undertow-jdk17-pod) -- jcmd 1 VM.info | grep -A13 '(cgroup)'
container (cgroup) information:
container_type: cgroupv2
cpu_cpuset_cpus: not supported
cpu_memory_nodes: not supported
active_processor_count: 3
cpu_quota: 300000
cpu_period: 100000
cpu_shares: 1024
memory_limit_in_bytes: 2147483648
memory_and_swap_limit_in_bytes: 2147483648
memory_soft_limit_in_bytes: unlimited
memory_usage_in_bytes: 181694464
memory_max_usage_in_bytes: not supported
```
