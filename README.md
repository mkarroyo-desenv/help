# instalando infra de nuvem em raspberries

Seguindo os passos/dicas do curso da Udemy Curso preparatório: Certificação CKA 
https://www.udemy.com/course/curso-preparatorio-certificacao-cka-kubernetes-v121/learn/lecture/35855172#overview

verificando a versão instalada:
```
cat /etc/os-release

PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

O professor utilizou Ubuntu 20.04 já que teve problemas com a versão 22.04 - pods ficavam instáveis, reiniciando a todo momento.


Acessando a documentação oficial:
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

A versão atual é a 1.30.

desabilitar o swap:
`sudo swapoff -a`

Se quiser visualizar.se o swap está habilitado, digitar `cat /etc/fstab`e verificar que não tem referência ao swap:
```
root@raspimaster:/etc/containerd# cat /etc/fstab
proc            /proc           proc    defaults          0       0
PARTUUID=c6d6fd0b-01  /boot/firmware  vfat    defaults          0       2
PARTUUID=c6d6fd0b-02  /               ext4    defaults,noatime  0       1
# a swapfile is not a swap partition, no line here
#   use  dphys-swapfile swap[on|off]  for that
```




Primeiro passo fazer a instalação do **container runtime!**
Seguindo a documentação oficial:
https://kubernetes.io/docs/setup/production-environment/container-runtimes/

Primeiramente carregar os módulos de rede!
```
sudo -i

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sysctl net.ipv4.ip_forward
```

Seguindo o roteiro, instalar o containerd. Existe o link para o Github:
https://github.com/containerd/containerd/blob/main/docs/getting-started.md

Existem três modos de realizar a inalação, fizemos utilizando o apt-get.
Escolhi a versão Debian e abrimos o link:
https://docs.docker.com/engine/install/debian/
e comecei seguindo o roteiro, desinstalando algum pacote velho/com conflito:
```
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

Em seguida começa a instalação em si:
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Na parte de instalação dos pacotes, só se faz necessário a instalação do containerd.io então digitei apenas:
`sudo apt-get install containerd.io`


OK, agora a gente volta para o tutorial anterior (Step 3: Installing CNI plugins https://github.com/containerd/containerd/blob/main/docs/getting-started.md
para a instalação do CNI.

Fazer o download da lista da versão (atualmente) 1.5.1, utilizando para o isso o wget!
https://github.com/containernetworking/plugins/releases

`wget https://github.com/containernetworking/plugins/releases/download/v1.5.1/cni-plugins-linux-amd64-v1.5.1.tgz`


arquivo baixado!
```
...
HTTP request sent, awaiting response... 200 OK
Length: 48222725 (46M) [application/octet-stream]
Saving to: ‘cni-plugins-linux-amd64-v1.5.1.tgz’

cni-plugins-linux-amd64-v1.5.1.tgz          100%[=========================================================================================>]  45.99M  2.06MB/s    in 31s     

2024-07-09 20:36:39 (1.49 MB/s) - ‘cni-plugins-linux-amd64-v1.5.1.tgz’ saved [48222725/48222725]

root@raspimaster:~# ls -lhtr
total 46M
-rw-r--r-- 1 root root 46M Jun 17 16:51 cni-plugins-linux-amd64-v1.5.1.tgz
```

Voltando ao roteiro
https://github.com/containerd/containerd/blob/main/docs/getting-started.md
pede para checar o arquivo (`sha256sum`) e extrair o conteúdo no diretório `/opt/cni/bin`
```
$ mkdir -p /opt/cni/bin
$ tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.1.tgz 
./
./macvlan
./static
./vlan
./portmap
./host-local
./vrf
./bridge
./tuning
./firewall
./host-device
./sbr
./loopback
./dhcp
./ptp
./ipvlan
./bandwidth
```

para testar se foi corretamente instalado:
```
root@raspimaster:~# systemctl status containerd
● containerd.service - containerd container runtime
     Loaded: loaded (/lib/systemd/system/containerd.service; enabled; preset: enabled)
     Active: active (running) since Tue 2024-07-09 01:57:17 BST; 18h ago
       Docs: https://containerd.io
    Process: 778 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 781 (containerd)
      Tasks: 9
        CPU: 648ms
     CGroup: /system.slice/containerd.service
             └─781 /usr/bin/containerd

Jul 09 01:57:17 raspimaster containerd[781]: time="2024-07-09T01:57:17.768080090+01:00" level=info msg="Start subscribing containerd event"
Jul 09 01:57:17 raspimaster containerd[781]: time="2024-07-09T01:57:17.768147331+01:00" level=info msg="Start recovering state"
Jul 09 01:57:17 raspimaster containerd[781]: time="2024-07-09T01:57:17.769723072+01:00" level=info msg="Start event monitor"
Jul 09 01:57:17 raspimaster containerd[781]: time="2024-07-09T01:57:17.769742442+01:00" level=info msg="Start snapshots syncer"
Jul 09 01:57:17 raspimaster containerd[781]: time="2024-07-09T01:57:17.769758331+01:00" level=info msg="Start cni network conf syncer for default"
Jul 09 01:57:17 raspimaster containerd[781]: time="2024-07-09T01:57:17.769770313+01:00" level=info msg="Start streaming server"
Jul 09 01:57:17 raspimaster containerd[781]: time="2024-07-09T01:57:17.769813146+01:00" level=info msg=serving... address=/run/containerd/containerd.sock.ttrpc
Jul 09 01:57:17 raspimaster containerd[781]: time="2024-07-09T01:57:17.769858683+01:00" level=info msg=serving... address=/run/containerd/containerd.sock
Jul 09 01:57:17 raspimaster containerd[781]: time="2024-07-09T01:57:17.769913776+01:00" level=info msg="containerd successfully booted in 0.062221s"
Jul 09 01:57:17 raspimaster systemd[1]: Started containerd.service - containerd container runtime.
root@raspimaster:~# 
```

Voltando para a documentação do Kubenetes
https://kubernetes.io/docs/setup/production-environment/container-runtimes/
temos que encontrar ou criar o arquivo `config.toml`, que no linux o padrão do diretório é `/etc/containerd/`
abrimos o arquivo (eu abri com o vi) e fui até o final do arquivo e incluí os seguintes dados:
```
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

O professor disse que tinha que achar a opção `disable_plugins = ["cri"]` e comentar essa linha mas no meu arquivo essa opção já estava em branco (`disable_plugins = []`)

Depois reiniciamos o containerd
`sudo systemctl restart containerd`
o meu deu erro e tive que ver o erro no journal. utilizei o comando `journalctl -xeu containerd.service`e no meio dos logs achei:
`Jul 09 20:54:52 raspimaster containerd[6666]: containerd: failed to load TOML: /etc/containerd/config.toml: (294, 3): duplicated tables`
Aí fui e editei novamente o arquivo `/etc/containerd/config.toml` e vi que os dados já estavam lá:
```
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = "" 
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = true
```

apaguei as linhas adicionadas e o reiniciei (com sucesso) o containerd.

container runtime configurado, agora voltemos a configuração do kubeadm!
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

Agora vamos instalar

**kubeadm** - responsável para criar os clusters
**kubelet** - será o agente dos nós, através do agente o cluster vai entender se os nós estão respondendo corretamente
**kubectl** - tem a interação com o kubenetes. **só se faz necessário instalar no nó master!**

seguindo o procedimento:
```
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

**NOTA** - sugestão/dica do professor, verificar qual é a versão e especificar na hora da instalação.
Para verificar digitar `apt list -a kubeadm`e é mostado a listagem a seguir:
```
apt list -a kubeadm
Listing... Done
kubeadm/unknown 1.30.2-1.1 amd64
kubeadm/unknown 1.30.1-1.1 amd64
kubeadm/unknown 1.30.0-1.1 amd64

kubeadm/unknown 1.30.2-1.1 arm64 [upgradable from: 1.28.11-1.1]
kubeadm/unknown 1.30.1-1.1 arm64
kubeadm/unknown 1.30.0-1.1 arm64
kubeadm/now 1.28.11-1.1 arm64 [installed,upgradable to: 1.30.2-1.1]

kubeadm/unknown 1.30.2-1.1 ppc64el
kubeadm/unknown 1.30.1-1.1 ppc64el
kubeadm/unknown 1.30.0-1.1 ppc64el

kubeadm/unknown 1.30.2-1.1 s390x
kubeadm/unknown 1.30.1-1.1 s390x
kubeadm/unknown 1.30.0-1.1 s390x
```

aí foi instalado a última versão estável.
No nosso caso o comando seria assim:
`apt-get install kubeadm=1.30.2-1.1`

a instalação padrão que eu tinha feito, ele instalou a versão 1.28.11, que nem está na lista. Atualizei e ficou assim:
```
root@raspimaster:/etc/containerd# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"28", GitVersion:"v1.28.11", GitCommit:"f25b321b9ae42cb1bfaa00b3eec9a12566a15d91", GitTreeState:"clean", BuildDate:"2024-06-11T20:18:34Z", GoVersion:"go1.21.11", Compiler:"gc", Platform:"linux/arm64"}

root@raspimaster:/etc/containerd# apt-get install kubeadm=1.30.2-1.1 
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  cri-tools
The following held packages will be changed:
  kubeadm
The following packages will be upgraded:
  cri-tools kubeadm
2 upgraded, 0 newly installed, 0 to remove and 3 not upgraded.
Need to get 28.4 MB of archives.
After this operation, 5,970 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  cri-tools 1.30.0-1.1 [19.4 MB]
Get:2 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  kubeadm 1.30.2-1.1 [8,910 kB]                                                  
Fetched 28.4 MB in 36s (783 kB/s)                                                                                                                                            
Reading changelogs... Done
(Reading database ... 56388 files and directories currently installed.)
Preparing to unpack .../cri-tools_1.30.0-1.1_arm64.deb ...
Unpacking cri-tools (1.30.0-1.1) over (1.28.0-1.1) ...
Preparing to unpack .../kubeadm_1.30.2-1.1_arm64.deb ...
Unpacking kubeadm (1.30.2-1.1) over (1.28.11-1.1) ...
Setting up cri-tools (1.30.0-1.1) ...
Setting up kubeadm (1.30.2-1.1) ...

root@raspimaster:/etc/containerd# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"30", GitVersion:"v1.30.2", GitCommit:"39683505b630ff2121012f3c5b16215a1449d5ed", GitTreeState:"clean", BuildDate:"2024-06-11T20:27:59Z", GoVersion:"go1.22.4", Compiler:"gc", Platform:"linux/arm64"}
```

Agora vamos para a documentação de como criar um cluster usando o kubeadm
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

Professor sugeriu baixar as imagens (dockers? containers?) básicas que farão o Kubernetes funcionar.
para saber os comandos possíveis do kubeadm, utilizar o help `-h`:
`kubeadm -h`
precisamos do config então
`kubeadm config -h`
queremos baixar as imagens
`kubeadm config images -h`
e existem as opções de `list` ou `pull`. Será o `pull` e rodei o comando (demorou um pouco para começar a baixar!):
```
root@raspimaster:/etc/containerd# kubeadm config images pull
[config/images] Pulled registry.k8s.io/kube-apiserver:v1.30.2
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.30.2
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.30.2
[config/images] Pulled registry.k8s.io/kube-proxy:v1.30.2
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.11.1
[config/images] Pulled registry.k8s.io/pause:3.9
  [config/images] Pulled registry.k8s.io/etcd:3.5.12-0
root@raspimaster:/etc/containerd#   
```

Para iniciar o container, o professor também recomenda sempre passar a versão utilizada (caso queira uma versão específica).
`kubeadm init -h`
`kubeadm init --kubernetes-version=1.30.0`
e aí deu erro :õ
```
root@raspimaster:/etc/containerd# kubeadm init --kubernetes-version=1.30.0
[init] Using Kubernetes version: v1.30.0
[preflight] Running pre-flight checks
[preflight] The system verification failed. Printing the output from the verification:
KERNEL_VERSION: 6.6.31+rpt-rpi-2712
CONFIG_NAMESPACES: enabled
CONFIG_NET_NS: enabled
CONFIG_PID_NS: enabled
CONFIG_IPC_NS: enabled
CONFIG_UTS_NS: enabled
CONFIG_CGROUPS: enabled
CONFIG_CGROUP_CPUACCT: enabled
CONFIG_CGROUP_DEVICE: enabled
CONFIG_CGROUP_FREEZER: enabled
CONFIG_CGROUP_PIDS: enabled
CONFIG_CGROUP_SCHED: enabled
CONFIG_CPUSETS: enabled
CONFIG_MEMCG: enabled
CONFIG_INET: enabled
CONFIG_EXT4_FS: enabled
CONFIG_PROC_FS: enabled
CONFIG_NETFILTER_XT_TARGET_REDIRECT: enabled (as module)
CONFIG_NETFILTER_XT_MATCH_COMMENT: enabled (as module)
CONFIG_FAIR_GROUP_SCHED: enabled
CONFIG_OVERLAY_FS: enabled (as module)
CONFIG_AUFS_FS: not set - Required for aufs.
CONFIG_BLK_DEV_DM: enabled (as module)
CONFIG_CFS_BANDWIDTH: enabled
CONFIG_CGROUP_HUGETLB: not set - Required for hugetlb cgroup.
CONFIG_SECCOMP: enabled
CONFIG_SECCOMP_FILTER: enabled
OS: Linux
CGROUPS_CPU: enabled
CGROUPS_CPUSET: enabled
CGROUPS_DEVICES: enabled
CGROUPS_FREEZER: enabled
CGROUPS_MEMORY: missing
CGROUPS_PIDS: enabled
CGROUPS_HUGETLB: missing
CGROUPS_IO: enabled
	[WARNING SystemVerification]: missing optional cgroups: hugetlb
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR SystemVerification]: missing required cgroups: memory
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```


Seguindo a dica do link
https://askubuntu.com/questions/1189480/raspberry-pi-4-ubuntu-19-10-cannot-enable-cgroup-memory-at-boostrap


Falava para editar o arquivo `/boot/firmware/cmdline.txt` (fiz uma cópia do arquivo antes) e adicionar os comandos `cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1`

Meu arquivo ela estava 
`console=serial0,115200 console=tty1 root=PARTUUID=c6d6fd0b-02 rootfstype=ext4 fsck.repair=yes rootwait cfg80211.ieee80211_regdom=BR


Adicionei no final do arquivo e reiniciei o raspi. E ele não ligou mais! Kkkkkkk brincadeirinha!
Loguei novamente, virei sudo e digitei apenas kubeadm init. Demorou um pouco (até parece que travou no passo que dizia que podia levar até 4 minutos!)
`[kubelet-check] Waiting for a healthy kubelet. This can take up to 4m0s`

E deu erro novamente! :/
```
[kubelet-check] Waiting for a healthy kubelet. This can take up to 4m0s
[kubelet-check] The kubelet is not healthy after 4m0.000782489s

Unfortunately, an error has occurred:
	The HTTP call equal to 'curl -sSL http://127.0.0.1:10248/healthz' returned error: Get "http://127.0.0.1:10248/healthz": context deadline exceeded


This error is likely caused by:
	- The kubelet is not running
	- The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
	- 'systemctl status kubelet'
	- 'journalctl -xeu kubelet'

Additionally, a control plane component may have crashed or exited when started by the container runtime.
To troubleshoot, list all containers using your preferred container runtimes CLI.
Here is one example how you may list all running Kubernetes containers by using crictl:
	- 'crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps -a | grep kube | grep -v pause'
	Once you have found the failing container, you can inspect its logs with:
	- 'crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock logs CONTAINERID'
error execution phase wait-control-plane: could not initialize a Kubernetes cluster
To see the stack trace of this error execute with --v=5 or higher
```

Lá fui eu futucar e o kubelet realmente não estava rodando:
```
root@raspimaster:~# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: activating (auto-restart) (Result: exit-code) since Wed 2024-07-10 22:09:56 BST; 429ms ago
       Docs: https://kubernetes.io/docs/
    Process: 1438 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=1/FAILURE)
   Main PID: 1438 (code=exited, status=1/FAILURE)
        CPU: 104ms
```

No journal aparecia a seguinte mensagem:
```
journalctl -xeu kubelet
…
Jul 10 22:10:47 raspimaster kubelet[1491]: I0710 22:10:47.510141    1491 server.go:725] "--cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /"
Jul 10 22:10:47 raspimaster kubelet[1491]: E0710 22:10:47.510262    1491 run.go:74] "command failed" err="failed to run Kubelet: running with swap on is not supported, pleas>
Jul 10 22:10:47 raspimaster systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
```

Como assim o swap está habilitado?
E não é que estava?
Vi nesse fórum que com o reboot o raspian habilitava o swap novamente.
https://forums.raspberrypi.com/viewtopic.php?t=244130


Entre as diversas dicas (editar arquivo,  adicionar o comando na crontab etc), optei por desabilitar o serviço usando o comando abaixo:
`sudo systemctl disable dphys-swapfile.service`

Reiniciei e confirmei (com o simples top) que o swap estava desligado.
Kubeadm aqui vamos nós novamente!
Para mais um novo erro!

```
[init] Using Kubernetes version: v1.30.2
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: missing optional cgroups: hugetlb
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR Port-6443]: Port 6443 is in use
	[ERROR Port-10259]: Port 10259 is in use
	[ERROR Port-10257]: Port 10257 is in use
	[ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: /etc/kubernetes/manifests/kube-apiserver.yaml already exists
	[ERROR FileAvailable--etc-kubernetes-manifests-kube-controller-manager.yaml]: /etc/kubernetes/manifests/kube-controller-manager.yaml already exists
	[ERROR FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml]: /etc/kubernetes/manifests/kube-scheduler.yaml already exists
	[ERROR FileAvailable--etc-kubernetes-manifests-etcd.yaml]: /etc/kubernetes/manifests/etcd.yaml already exists
	[ERROR Port-10250]: Port 10250 is in use
	[ERROR Port-2379]: Port 2379 is in use
	[ERROR Port-2380]: Port 2380 is in use
	[ERROR DirAvailable--var-lib-etcd]: /var/lib/etcd is not empty
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```




Nova pesquisa de fóruns e achei essa dica salvadora:
https://discuss.kubernetes.io/t/kubeadm-init-fails/16888/13
`kubeadm reset`

Reiniciei e rodei novamente kubeamd init como super usuário.
FINALMENTE CONSEGUI! \o/
```
user@raspimaster:~ $ sudo -i
root@raspimaster:~# kubeadm init
[init] Using Kubernetes version: v1.30.2
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: missing optional cgroups: hugetlb
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W0711 00:23:09.674568     919 checks.go:844] detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm.It is recommended to use "registry.k8s.io/pause:3.9" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local raspimaster] and IPs [10.96.0.1 192.168.1.169]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost raspimaster] and IPs [192.168.1.169 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost raspimaster] and IPs [192.168.1.169 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[kubelet-check] Waiting for a healthy kubelet. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 1.500949079s
[api-check] Waiting for a healthy API server. This can take up to 4m0s
[api-check] The API server is healthy after 10.001627864s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node raspimaster as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node raspimaster as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: sqi9uq.u8by4wgpg2l67gys
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.169:6443 --token sqi9uq.u8by4wgpg2l67gys \
	--discovery-token-ca-cert-hash sha256:d92cfe93fe9c6958da6929050997c0af3abb658753ccfe1ed527b4c77e8dd98d 
root@raspimaster:~# 
```

Como dizia no final da instalação do kubeadm, preciso executar esses comandos: (como usuário comum):
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

E, para testar se está tudo ok, listo os nós:
```
user@raspimaster:~ $ kubectl get nodes
NAME          STATUS     ROLES           AGE   VERSION
raspimaster   NotReady   control-plane   15h   v1.28.11
```

 NOTA: Antes de criar o cluster deveria ter sido feito a troca do nome do hostname, para ficar “bonitinho/padronizado”. Para fazer isso um dos métodos seria editar o arquivo `/etc/hostname` aí tem que reverter o kubeadm com o comando kubeadm reset` e reinstalar o kubeadmin com o comando `kubeadmin init`

NOTA2: verificamos que fo instalado a versão 1.28.11 do control-lane!


Agora precisa instalar o plugin de rede.
Na documentação oficial procurar por addons e nos leva a página
https://kubernetes.io/docs/concepts/cluster-administration/addons/

Foi sugerido instalar o Waeave Net. Havia reclamação que a página não funcionava (na minha instalação a página do Weave Net foi desligada mas havia o link/repsoitório para o GitHub) então foi sugerido usar o Calico.

Waeave Net
https://github.com/rajch/weave#using-weave-on-kubernetes

Para instalar basta fazer o apply do pacote:
`kubectl delete -n kube-system -f weave-net.yaml`

Em um comparativo entre Calico, Flannel, Weave e Cilium, o Calico é mais performático e tem maiores opções de segurança.
https://www.devopsschool.com/blog/compare-the-differences-between-calico-flannel-weave-and-cilium/

Caso tenha instalado o Weavenet e queira trocar pelo Calico, aqui tem um roteiro de como fazer a troca:
https://medium.com/@andresperezl/weave-to-calico-7c69677222d0


Eu optei por instalar o Calico e segui o seguinte tutorial:
https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

Instalando:
```
user@raspimaster:~ $ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
namespace/tigera-operator created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/apiservers.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/imagesets.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/installations.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/tigerastatuses.operator.tigera.io created
serviceaccount/tigera-operator created
clusterrole.rbac.authorization.k8s.io/tigera-operator created
clusterrolebinding.rbac.authorization.k8s.io/tigera-operator created
deployment.apps/tigera-operator created
user@raspimaster:~ $ 

user@raspimaster:~ $ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
installation.operator.tigera.io/default created
apiserver.operator.tigera.io/default created
user@raspimaster:~ $
```


Após a instalação, visualizando os nodes verá que agora o control-plane está pronto:
`kubectl get nodes`

Para mim essa configuração não deu certo. Pensa Márcia!
Listei os deploy e ele estava lá!
```
user@raspimaster:~ $ kubectl get deploy -A
NAMESPACE         NAME              READY   UP-TO-DATE   AVAILABLE   AGE
kube-system       coredns           0/2     2            0           16h
tigera-operator   tigera-operator   1/1     1            1           8m4s
user@raspimaster:~ $ 
```

Listei os pods rodando, e ele estava lá!
```
user@raspimaster:~ $ kubectl get pods -A
NAMESPACE         NAME                                  READY   STATUS    RESTARTS       AGE
kube-system       coredns-7db6d8ff4d-jjwbh              0/1     Pending   0              16h
kube-system       coredns-7db6d8ff4d-t68th              0/1     Pending   0              16h
kube-system       etcd-raspimaster                      1/1     Running   10 (15h ago)   16h
kube-system       kube-apiserver-raspimaster            1/1     Running   8 (15h ago)    16h
kube-system       kube-controller-manager-raspimaster   1/1     Running   11 (15h ago)   16h
kube-system       kube-proxy-rq9kl                      1/1     Running   1 (15h ago)    16h
kube-system       kube-scheduler-raspimaster            1/1     Running   9 (15h ago)    16h
tigera-operator   tigera-operator-76ff79f7fd-52c5d      1/1     Running   0              8m36s
user@raspimaster:~ $
```

Aí fui tentar entender o motivo visualizando os eventos:
```
user@raspimaster:~ $ kubectl get events -n kube-system | grep -v Normal | sort -h
LAST SEEN   TYPE      REASON              OBJECT                                    MESSAGE
4m34s       Warning   DNSConfigForming    pod/kube-scheduler-raspimaster            Nameserver limits were exceeded, some nameservers have been omitted, the applied nameserver line is: 8.8.8.8 8.8.4.4 2804:14d:1:0:181:213:132:2
4m35s       Warning   DNSConfigForming    pod/kube-apiserver-raspimaster            Nameserver limits were exceeded, some nameservers have been omitted, the applied nameserver line is: 8.8.8.8 8.8.4.4 2804:14d:1:0:181:213:132:2
4m37s       Warning   DNSConfigForming    pod/kube-proxy-rq9kl                      Nameserver limits were exceeded, some nameservers have been omitted, the applied nameserver line is: 8.8.8.8 8.8.4.4 2804:14d:1:0:181:213:132:2
5m14s       Warning   FailedScheduling    pod/coredns-7db6d8ff4d-jjwbh              0/1 nodes are available: 1 node(s) had untolerated taint {node.kubernetes.io/not-ready: }. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
5m14s       Warning   FailedScheduling    pod/coredns-7db6d8ff4d-t68th              0/1 nodes are available: 1 node(s) had untolerated taint {node.kubernetes.io/not-ready: }. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
20s         Warning   DNSConfigForming    pod/etcd-raspimaster                      Nameserver limits were exceeded, some nameservers have been omitted, the applied nameserver line is: 8.8.8.8 8.8.4.4 2804:14d:1:0:181:213:132:2
9s          Warning   DNSConfigForming    pod/kube-controller-manager-raspimaster   Nameserver limits were exceeded, some nameservers have been omitted, the applied nameserver line is: 8.8.8.8 8.8.4.4 2804:14d:1:0:181:213:132:2
```

Novas pesquisa me levou para essa dica (https://discuss.kubernetes.io/t/0-1-nodes-are-available-1-node-s-had-untolerated-taint/23248), de que faltou um passo:
Documentação oficial (https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation) diz que, por segurança, o cluster não vai agendar/rodar pods no nó do control-plane. Para desabilitar usar o comando abaixo:

user@raspimaster:~ $ kubectl taint nodes --all node-role.kubernetes.io/control-plane-
node/raspimaster untainted
user@raspimaster:~ $ kubectl get nodes
NAME          STATUS     ROLES           AGE   VERSION
raspimaster   NotReady   control-plane   16h   v1.28.11
user@raspimaster:~ $ kubectl get pods -A
NAMESPACE         NAME                                  READY   STATUS    RESTARTS       AGE
kube-system       coredns-7db6d8ff4d-jjwbh              0/1     Pending   0              16h
kube-system       coredns-7db6d8ff4d-t68th              0/1     Pending   0              16h
kube-system       etcd-raspimaster                      1/1     Running   10 (16h ago)   16h
kube-system       kube-apiserver-raspimaster            1/1     Running   8 (16h ago)    16h
kube-system       kube-controller-manager-raspimaster   1/1     Running   11 (16h ago)   16h
kube-system       kube-proxy-rq9kl                      1/1     Running   1 (16h ago)    16h
kube-system       kube-scheduler-raspimaster            1/1     Running   9 (16h ago)    16h
tigera-operator   tigera-operator-76ff79f7fd-52c5d      1/1     Running   0              17m



Fui reverificar o fiz o seguinte:
```
kubectl config view
apiVersion: v1
clusters: null
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
root@raspimaster:~
```

O arquivo estava estranho. Verifiquei o arquivo admin.conf no diretório `cat /etc/kubernetes/admin.conf` e ele estava direito então refiz o penúltimo passo da instalação do kubeadm, como usuário normal:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


Aí reconheci o arquivo e estava “normal”:
```
kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.1.169:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
user@raspimaster:~ $
```


https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day-5/README.md#instalando-o-kubeadm
kubeadm join 192.168.1.169:6443 --token vewfhz.43bnacd0riu0g7r1 \
	--discovery-token-ca-cert-hash sha256:e39f80f6c018615834cd2cab3084bec57a98d4d647e786312ce9fd00d3034f0a 








Então voltei a tentativa de instalação do calico pelas instruções do link
```
kubectl replace -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl replace -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```



https://stackoverflow.com/questions/52609257/coredns-in-pending-state-in-kubernetes-cluster
Antes de instalar o calico, zerar o kubeadm e recomeçar com essas dicas
`kubeadm reset`
--pod-network-cidr
Kubeadmin init --pod-network-cidr=10.244.0.0/16

tempo 27:44 minutos
