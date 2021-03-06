.. _kubeadm:

===========
kubeadm
===========

kubeadm是用于自举kubernetes集群的工具，我们将使用kubeadm结合kubelet/kubectl来构建一个Kubernetes集群。

准备工作
==========

- 硬件: 2CPU / 2GB内存 ，并且服务器必须 ``禁用`` swap，否则kubelet可能工作不正常（kubeadm初始化系统时会检查swap，如存在swap会报错）

.. note::

   批量关闭所有Kubernetes服务器上swap::

      pssh -ih kube 'sudo swapoff -a'
      pssh -ih kube "sudo sed -i -e '/swap/s/^UUID=/#UUID=/g' /etc/fstab" 

- 网络：每个主机具有唯一主机名，MAC地址以及 ``product_uuid`` ，并且开放必要管控端口(见下文)
- 操作系统：采用符合要求OS，通常 Ubuntu 16.04+, CentOS 7, Fedora 25/26

验证MAC地址和product_uuid
----------------------------

- 检查MAC地址::

   ip link

- 检查 ``product_uuid`` ::

   sudo cat /sys/class/dmi/id/product_uuid

.. note::

   务必确保服务器具有唯一地址，特别是一些虚拟机可能会使用重复的标识值，会导致Kubernetes的安装过程失败。我在 :ref:`k8s_hosts` 采用KVM虚拟机是通过 ``virt-clone`` 和 ``virt-sysprep`` 构建的，会自动初始化不同的 ``product_uuid`` 和 MAC地址。

检查网络
------------

- 对于多网卡环境，特别是Kubernetes组件不是通过默认路由访问，建议添加IP路由以便Kubernetes集群地址能够通过相应网卡互访

网络端口
~~~~~~~~~~

- 管控平台节点:

====== ====== =========== =======================  ======================
协议   方向   端口范围    用途                     使用方
====== ====== =========== =======================  ======================
TCP    入     6443*       Kubernetes API服务器     All
TCP    入     2379-2380   etcd服务器客户端访问API  kube-apiserver, etcd
TCP    入     10250       Kubelet API              自身，管控平台
TCP    入     10251       kube-scheduler           自身
TCP    入     10252       kube-controller-manager  自身
====== ====== =========== =======================  ======================

- 工作节点:

====== ====== =========== =======================  ======================
协议   方向   端口范围    用途                     使用方
====== ====== =========== =======================  ======================
TCP    入     10250       Kubelet API              自身，管控平台
TCP    入     30000-32767 节点端口服务             All
====== ====== =========== =======================  ======================

.. note::

   由于虚拟机安装默认开启了firewalld服务，所以我批量通过以下命令来开启端口::

      pssh -ih kubemaster 'sudo firewall-cmd --zone=public --add-port=6443/tcp --permanent'
      pssh -ih kubemaster 'sudo firewall-cmd --zone=public --add-port=2379-2380/tcp --permanent'
      pssh -ih kubemaster 'sudo firewall-cmd --zone=public --add-port=10250-10252/tcp --permanent'

      pssh -ih kubenode 'sudo firewall-cmd --zone=public --add-port=10250/tcp --permanent'
      pssh -ih kubenode 'sudo firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent'

安装runtime
=============

从 v1.6.0 开始，Kuberneetes激活默认使用CRI, Container Runtime

从 v1.14.0 开始， kubeadm 将自动检测Linux节点的容器运行环境，这个检测是基于总所周知的domain sockets。

=========== ===================================
Runtime     Domain Socket
=========== ===================================
Docker      /var/run/docker.sock
contained   /run/containerd/containerd.sock
CRI0-O      /var/run/crio/crio.sock
=========== ===================================

如果同时检测到 Docker 和 containerd ，则优先采用Docker。因为Docker 18.09同时提供了containerd，所以两者都会被检测到。

对于非Linux节点，则默认runtime是Docker。

安装kubeadm, kubelet 和 kubectl
==================================

在所有节点上需要安装以下软件包:

- ``kubeadm`` 启动cluster的命令工具
- ``kubelet`` 运行在集群所有服务器节点的组件，用于启动pod和容器
- ``kubectl`` 和集群通讯的工具

由于 kubeadm ``不会`` 安装和管理 ``kubelet`` 或 ``kubectl`` ，所以要确保你需要kubeadm为你安装的正确版本。

.. _prepare_for_k8s_install:

安装kubernetes前准备
----------------------

- 请参考 :ref:`openconnect_vpn` 准备好梯子，安装Kubernetes软件包需要访问Google的软件仓库。

注意：请不要直接在Kubernetes集群服务器上部署VPN客户端来翻墙，我遇到的问题是，VPN客户端运行时会在服务器上添加 ``tun0`` 网络设备，并且设置了默认路由。这会导致 ``kubeadm`` 初始化时以 ``tun0`` 接口的IP地址作为API服务接口。这样一旦关闭VPN接口就会引起异常。

解决的方法时采用VPN网关方式，在外部的服务器上构建VPN转发，这样局域网内部的服务器就不需要单独运行VPN，也就不再为这个网卡无识别困扰了。 具体方法请参考 :ref:`polipo_proxy_yum`

如果部署Kubernetes时使用了VPN，导致VPN关闭时无法运行Kubernetes，则需要 :ref:`change_master_ip` 来恢复正确配置。

- 节点上的 SELinux 需要设置成 ``permissive`` 模式::

   setenforce 0
   sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

这样容器才可以访问主机的文件系统，这个特性是pod网络锁需要的特性。目前kubelet还不能支持SELinux，所以需要禁用SELinux。

- 一些RHEL/CentOS 7用户报告流量路由错误，因为iptables被绕过。所以需要确保 ``net.bridge.bridge-nf-call-iptables`` 设置成 ``1`` ::

   cat <<EOF >  /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   EOF
   sysctl --system

该步骤必须执行，否则 ``kubedam init`` 时报错。
  
- 确保先加载 ``br_netfilter`` 模块已经加载，通过 ``lsmod | grep br_netfilter`` 确保，如果没有加载，则执行以下命令加载该内核模块::

   modprobe br_netfilter

``kubelet`` 会不断重启，以等待kubeadm的crashloop告知其执行。

.. note::

   批量处理方式::

      pssh -ih kube "sudo setenforce 0;sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config"

      cat <<EOF > k8s._onf
      net.bridge.bridge-nf-call-ip6tables = 1
      net.bridge.bridge-nf-call-iptables = 1
      EOF

      pscp.pssh -h kube k8s.conf /tmp/k8s.conf
      pssh -ih kube "sudo cp /tmp/k8s.conf /etc/sysctl.d/k8s.conf"
      pssh -ih kube "sudo sysctl --system"


Ubuntu, Debian
----------------

::

   apt-get update && apt-get install -y apt-transport-https curl
   curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
   cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
   deb https://apt.kubernetes.io/ kubernetes-xenial main
   EOF
   apt-get update
   apt-get install -y kubelet kubeadm kubectl
   apt-mark hold kubelet kubeadm kubectl

CentOS, RHEL, Fedora
------------------------

::

   cat <<EOF > /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=1
   repo_gpgcheck=1
   gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
   EOF
   
   # Set SELinux in permissive mode (effectively disabling it)
   setenforce 0
   sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
   
   yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
   
   systemctl enable --now kubelet

.. note::

   由于Kubernetes软件仓库由 "不存在公司" Google提供，所以需要 **翻墙** `飞越疯人院 <https://movie.douban.com/subject/1292224/>`_ ，请参考 :ref:`openconnect_vpn` 安装。

   对于采用NAT模式的KVM虚拟机集群，只需要在Host物理主机上启用VPN客户端就可以使得各节点获得正常的Internet访问。

.. note::

   批量处理命令::

      cat <<EOF > kubernetes.repo
      cat <<EOF > kubernetes.repo
      [kubernetes]
      name=Kubernetes
      baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
      enabled=1
      gpgcheck=1
      repo_gpgcheck=1
      gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
      EOF

      pscp.pssh -h kube kubernetes.repo /tmp/kubernetes.repo
      pssh -ih kube 'sudo cp /tmp/kubernetes.repo /etc/yum.repos.d/kubernetes.repo'

      pssh -ih kube 'sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes'
      pssh -ih kube 'sudo systemctl enable --now kubelet'

在管控平台节点配置kubelet使用cgroup driver
=============================================

.. note::

   这个步骤暂时不需要执行，因为在下一步 ``kubeadm init`` 时会初始化环境并自动配置并启动kubelet。

当在使用Docker的环境中，kubeadm可以为kubelet自动检测到cgroup driver，并在运行时设到 ``/var/lib/kubelet/kubeadm-flags.env`` 。

.. note::

   在使用Docker环境中，kubelet设置 ``/var/lib/kubelet/kubeadm-flags.env`` 如下::

      KUBELET_KUBEADM_ARGS="--cgroup-driver=systemd --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1"

   这个文件是 ``kubeadmin init`` 和 ``kubeadm join`` 时使用。

不过对于其他CRI，则需要修改 ``/etc/default/kubelet`` 设置 ``cgroup-driver`` 值，类似::

   KUBELET_EXTRA_ARGS=--cgroup-driver=<value>

这个文件在 ``kubeadm init`` 和 ``kubeadm join`` 到kubelet的用户定义参数扩展时使用。注意， **只有** CRI不是 ``cgroupfs`` 时才需要定义 cgroup driver。

然后重启 kubelet ::

   systemctl daemon-reload
   systemctl restart kubelet

.. note::

   这里还没有初始化集群，无法启动kubelet，出现报错::

      Jul 29 17:25:11 devstack kubelet[10529]: F0729 17:25:11.339363   10529 server.go:198] failed to load Kubelet config file /var/lib/kubelet/config.yaml, error failed to read kubelet config file "/var/lib/kubelet/config.yaml", error: open /var/lib/kubelet/config.yaml: no such file or directory

   通过 ``kubeadm init --pod-network-cidr=10.244.0.0/16`` 初始化集群。 见 :ref:`single_master_k8s`

kubelet排查(待续)
===================

.. note::

   目前这个 ``cpu and memory cgroup hierarchy not unified`` 问题尚未解决，不过我仅在 `AliOS 转换CentOS <https://github.com/huataihuang/cloud-atlas-draft/blob/master/os/linux/redhat/package/convert_alios_to_centos.md>`_ 的系统中遇到，而纯净的CentOS部署Kubernetes则完全没有问题。虽然看上去两种操作系统的cgroup没有太大差别，但是我怀疑或许在转换中我有某些软件包转换问题或者AliOS原先的cgroup配置有什么坑在里面尚未发觉。

   这个问题留待后续再排查，目前我改为在纯净版CentOS中部署原生Kubernetes集群。

由于我使用标准的Docker，所以kubeadm会自动检测cgroup driver，并设置环境 ``/var/lib/kubelet/kubeadm-flags.env`` 所以我没有设置 ``KUBELET_EXTRA_ARGS`` 直接启动 ``kubelet`` 。但是启动失败，排查如下：

- ``systemctl status kubelet`` ::

   ● kubelet.service - kubelet: The Kubernetes Node Agent
      Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
     Drop-In: /usr/lib/systemd/system/kubelet.service.d
              └─10-kubeadm.conf
      Active: activating (auto-restart) (Result: exit-code) since Thu 2019-07-25 09:04:17 CST; 1s ago
        Docs: https://kubernetes.io/docs/
     Process: 86474 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=255)
    Main PID: 86474 (code=exited, status=255)

- ``journalctl -xeu kubelet`` ::

   Jul 25 09:10:46 worker1.sqa.ztt kubelet[92407]: I0725 09:10:46.981191   92407 server.go:791] Client rotation is on, will bootstrap in background
   Jul 25 09:10:46 worker1.sqa.ztt kubelet[92407]: I0725 09:10:46.984895   92407 certificate_store.go:129] Loading cert/key pair from "/var/lib/kubelet/pki/kubelet-client-current.pem".
   Jul 25 09:10:46 worker1.sqa.ztt kubelet[92407]: F0725 09:10:46.988391   92407 server.go:273] failed to run Kubelet: failed to get the kubelet's cgroup: cpu and memory cgroup hierarchy not unified.  cpu: /, memory: /system.slice/kubelet.service
   Jul 25 09:10:46 worker1.sqa.ztt systemd[1]: kubelet.service: main process exited, code=exited, status=255/n/a

这里提示 cpu 和 memory 的 cgroup层次结构不统一。从 CentOS 来看 ``/sys/fs/cgroup`` 目录下::

   lrwxrwxrwx 1 root root 18 Jul 24 01:02 cpu -> cpuset,cpu,cpuacct
   lrwxrwxrwx 1 root root 18 Jul 24 01:02 cpuacct -> cpuset,cpu,cpuacct
   lrwxrwxrwx 1 root root 18 Jul 24 01:02 cpuset -> cpuset,cpu,cpuacct
   ...
   drwxr-xr-x 5 root root  0 Jul 24 01:02 memory

.. note::

   参考 `社区设计文档：design-proposals > kubelet-systemd <https://stupefied-goodall-e282f7.netlify.com/contributors/design-proposals/node/kubelet-systemd/>`_  和 `Kubernetes问题调查：failed to get cgroup stats for /systemd/system.slice <https://www.lijiaocn.com/%E9%97%AE%E9%A2%98/2019/01/25/kubernetes-failed-to-get-cgroup-stats.html>`_ 思路排查。

在 ``pkg/kubelet/cm/container_manager_linux.go`` 中

.. code-block:: go

   // getContainer returns the cgroup associated with the specified pid.
   // It enforces a unified hierarchy for memory and cpu cgroups.
   // On systemd environments, it uses the name=systemd cgroup for the specified pid.
   func getContainer(pid int) (string, error) {
   	cgs, err := cgroups.ParseCgroupFile(fmt.Sprintf("/proc/%d/cgroup", pid))
   	if err != nil {
   		return "", err
   	}
   
   	cpu, found := cgs["cpu"]
   	if !found {
   		return "", cgroups.NewNotFoundError("cpu")
   	}
   	memory, found := cgs["memory"]
   	if !found {
   		return "", cgroups.NewNotFoundError("memory")
   	}
   
   	// since we use this container for accounting, we need to ensure its a unified hierarchy.
   	if cpu != memory {
   		return "", fmt.Errorf("cpu and memory cgroup hierarchy not unified.  cpu: %s, memory: %s", cpu, memory)
   	}
   
   	// on systemd, every pid is in a unified cgroup hierarchy (name=systemd as seen in systemd-cgls)
   	// cpu and memory accounting is off by default, users may choose to enable it per unit or globally.
   	// users could enable CPU and memory accounting globally via /etc/systemd/system.conf (DefaultCPUAccounting=true DefaultMemoryAccounting=true).
   	// users could also enable CPU and memory accounting per unit via CPUAccounting=true and MemoryAccounting=true
   	// we only warn if accounting is not enabled for CPU or memory so as to not break local development flows where kubelet is launched in a terminal.
   	// for example, the cgroup for the user session will be something like /user.slice/user-X.slice/session-X.scope, but the cpu and memory
   	// cgroup will be the closest ancestor where accounting is performed (most likely /) on systems that launch docker containers.
   	// as a result, on those systems, you will not get cpu or memory accounting statistics for kubelet.
   	// in addition, you would not get memory or cpu accounting for the runtime unless accounting was enabled on its unit (or globally).
   	if systemd, found := cgs["name=systemd"]; found {
   		if systemd != cpu {
   			klog.Warningf("CPUAccounting not enabled for pid: %d", pid)
   		}
   		if systemd != memory {
   			klog.Warningf("MemoryAccounting not enabled for pid: %d", pid)
   		}
   		return systemd, nil
   	}
   
   	return cpu, nil
   }

从上述代码分析，如果使用 ``systemd`` 来管理cgroup，每个pid都有一个唯一cgroup树结构。默认情况下，cpu和memory记账是关闭的，可以通过针对每个单元或者全局启用。例如全局启用，则修改 ``/etc/systemd/system.conf`` ::

   DefaultCPUAccounting=true
   DefaultMemoryAccounting=true

如果针对某个应用，例如 kubelet ，则配置 ::

   CPUAccounting=true
   MemoryAccounting=true

上述cgroup记账功能不影响kubelet运行，但是启动时会WARNING。

.. note::

   参考 `CentOS kubelet complains about systemd slices #440 <https://github.com/kontena/pharos-cluster/issues/440>`_ :
   
   The correct fix for this issue is to configure systemd to create the desired per-service CPU/Memory cgroups for the kubelet::
   
      [Service]
      CPUAccounting=true
      MemoryAccounting=true
   
   这个建议参考 `Kubelet 'failed to get cgroup stats for "/system.slice/kubelet.service"' error messages #4049 <https://github.com/kubernetes/kops/issues/4049>`_ 是修订 ``/etc/systemd/system/kubelet.service.d/11-cgroups.conf`` ::
   
      [Service]
      CPUAccounting=true
      MemoryAccounting=true
   
如果cpu的cgroup hierarchy 和 memory的cgroup hierarchy 不一致，则报错退出。

验证可以检查任何一个系统进程 ``cat /proc/<pid>/cgroup`` 都可以看到 cpu 和 memory 的cgroup hierarchy不同::

   #cat /proc/46006/cgroup
   ...
   9:memory:/system.slice/sshd.service
   ...
   5:cpuacct,cpu,cpuset:/
   ...

.. note::

   `unified-hierarchy <https://lwn.net/Articles/601923/>`_ 是下一代cgrooup v2接口的特性，请参考 `cgroupv2: Linux's new unified control group system <https://qconlondon.com/system/files/presentation-slides/cgroupv2-qcon.pdf>`_ 介绍。目前看，内核 4.6以上结合systemd v226以上能够实现这个特性。

参考
========

- `Installing kubeadm <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>`_
- `Troubleshooting kubeadm <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/>`_
