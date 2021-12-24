---
layout:     post
title:      "kubeadm init 源码解析"
description: "kubeadm init 源码解析"
date:     2021-12-23
author: sfeng
categories: ["Tech", "cloudNative", "dev"]
tags: ["kubernetes", "kubeadm"]
URL: "/2021/12/23/kubeadm-init/"
---

## 使用二进制部署k8s

在 `kubeadm` 没出现之前，部署 `kubernetes` 都是使用二进制部署的方式进行部署，步骤多且复杂，需要对 `kubernetes`本身有专业的理解。所以 `kubeadm` 的出现使得部署步骤精简许多，现在通过阅读源码的方式弄清楚 `kubeadm` 的原理。

大家在阅读以下内容时，最好有过使用二进制部署`kubernetes` 的经验，这样可以变阅读源码变结合二进制部署的步骤，使得更容易立即理解，因为 `kubeadm` 实际上就是将二进制部署的步骤封装实现了而已。

`kubeadm` 源码和 `kubernetes` 在同一个项目里，大家不要误以为是 https://github.com/kubernetes/kubeadm.git

### 说明

> 在使用 `kubeadm` 之前需要安装一些软件，`kubeadm` 才能执行部署，需要在每台机器上安装 `kubeadm` 和 `kubelet`，`kubectl` 是可选的。                                                                   `kubeadm` 需要使用 `rpm` 包或者`dpg` 包安装，因为 `kubeadm` 包封装了 `kubelet` 启动 `service` 文件，同样 `kubelet` 也需要使用 `rpm` 包或者 `dpg` 包安装。
>

这里以 `kubeadm v1.19.8` 版本为例说明

## kubeadm 命令

`kubernetes\cmd\kubeadm\app\cmd\cmd.go`

以下是 `kubeadm` 支持的命令，这篇文章主要讲解 `kubeadm init` 及 `kubeam join` 的实现。

```go
// NewKubeadmCommand returns cobra.Command to run kubeadm command
func NewKubeadmCommand(in io.Reader, out, err io.Writer) *cobra.Command {
	var rootfsPath string

	cmds := &cobra.Command{
		Use:   "kubeadm",
		Short: "kubeadm: easily bootstrap a secure Kubernetes cluster",
		Long: dedent.Dedent(`

			    ┌──────────────────────────────────────────────────────────┐
			    │ KUBEADM                                                  │
			    │ Easily bootstrap a secure Kubernetes cluster             │
			    │                                                          │
			    │ Please give us feedback at:                              │
			    │ https://github.com/kubernetes/kubeadm/issues             │
			    └──────────────────────────────────────────────────────────┘

			Example usage:

			    Create a two-machine cluster with one control-plane node
			    (which controls the cluster), and one worker node
			    (where your workloads, like Pods and Deployments run).

			    ┌──────────────────────────────────────────────────────────┐
			    │ On the first machine:                                    │
			    ├──────────────────────────────────────────────────────────┤
			    │ control-plane# kubeadm init                              │
			    └──────────────────────────────────────────────────────────┘

			    ┌──────────────────────────────────────────────────────────┐
			    │ On the second machine:                                   │
			    ├──────────────────────────────────────────────────────────┤
			    │ worker# kubeadm join <arguments-returned-from-init>      │
			    └──────────────────────────────────────────────────────────┘

			    You can then repeat the second step on as many other machines as you like.

		`),
		SilenceErrors: true,
		SilenceUsage:  true,
		PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
			if rootfsPath != "" {
				if err := kubeadmutil.Chroot(rootfsPath); err != nil {
					return err
				}
			}
			return nil
		},
	}

	cmds.ResetFlags()
	# 以下是kubeadm 所支持的命令
	cmds.AddCommand(NewCmdCompletion(out, ""))
	cmds.AddCommand(NewCmdConfig(out))
	cmds.AddCommand(NewCmdInit(out, nil))
	cmds.AddCommand(NewCmdJoin(out, nil))
	cmds.AddCommand(NewCmdReset(in, out, nil))
	cmds.AddCommand(NewCmdVersion(out))
	cmds.AddCommand(NewCmdToken(out, err))
	cmds.AddCommand(upgrade.NewCmdUpgrade(out))
	cmds.AddCommand(alpha.NewCmdAlpha(in, out))

	options.AddKubeadmOtherFlags(cmds.PersistentFlags(), &rootfsPath)

	return cmds
}
```

## 命令行解析
kubeadm init 支持传递很多参数来自定义参数

```go
func NewCmdInit(out io.Writer, initOptions *initOptions) *cobra.Command {
  // 因为上面 initOptions 传递的值为 nil，所以会执行 newInitOptions()，我们主要看这个函数做些什么
	if initOptions == nil {
		initOptions = newInitOptions()
	}
	initRunner := workflow.NewRunner()

	cmd := &cobra.Command{
		Use:   "init",
		Short: "Run this command in order to set up the Kubernetes control plane",
		RunE: func(cmd *cobra.Command, args []string) error {
			c, err := initRunner.InitData(args)
			if err != nil {
				return err
			}

			data := c.(*initData)
			fmt.Printf("[init] Using Kubernetes version: %s\n", data.cfg.KubernetesVersion)

			if err := initRunner.Run(args); err != nil {
				return err
			}

			return showJoinCommand(data, out)
		},
		Args: cobra.NoArgs,
	}
```

这个函数会设置一些默认参数

- InitConfiguration
- ClusterConfiguration
- BootstrapToken
- kubeconfig Dir
- uploadCerts

```go
// newInitOptions returns a struct ready for being used for creating cmd init flags.
func newInitOptions() *initOptions {
	// initialize the public kubeadm config API by applying defaults
	externalInitCfg := &kubeadmapiv1beta2.InitConfiguration{}
	kubeadmscheme.Scheme.Default(externalInitCfg)

	externalClusterCfg := &kubeadmapiv1beta2.ClusterConfiguration{}
	kubeadmscheme.Scheme.Default(externalClusterCfg)

	// 创建默认token，这个 token 用于增加节点时，临时向apiserver注册的token，有效期为·24h,切加入默认用户组：system:bootstrappers:kubeadm:default-node-token
	bto := options.NewBootstrapTokenOptions()
	bto.Description = "The default bootstrap token generated by 'kubeadm init'."

	return &initOptions{
		externalInitCfg:    externalInitCfg,
		externalClusterCfg: externalClusterCfg,
		bto:                bto,
		kubeconfigDir:      kubeadmconstants.KubernetesDir,
		kubeconfigPath:     kubeadmconstants.GetAdminKubeConfigPath(),
		uploadCerts:        false,
	}
}
```

## kubeadm init

安装集群时，`kubeadm` 会依次执行以下步骤

`kubernetes\cmd\kubeadm\app\cmd\init.go`

```go
// initialize the workflow runner with the list of phases
	initRunner.AppendPhase(phases.NewPreflightPhase())
	initRunner.AppendPhase(phases.NewCertsPhase())
	initRunner.AppendPhase(phases.NewKubeConfigPhase())
	initRunner.AppendPhase(phases.NewKubeletStartPhase())
	initRunner.AppendPhase(phases.NewControlPlanePhase())
	initRunner.AppendPhase(phases.NewEtcdPhase())
	initRunner.AppendPhase(phases.NewWaitControlPlanePhase())
	initRunner.AppendPhase(phases.NewUploadConfigPhase())
	initRunner.AppendPhase(phases.NewUploadCertsPhase())
	initRunner.AppendPhase(phases.NewMarkControlPlanePhase())
	initRunner.AppendPhase(phases.NewBootstrapTokenPhase())
	initRunner.AppendPhase(phases.NewKubeletFinalizePhase())
	initRunner.AppendPhase(phases.NewAddonPhase())
```

### **Prefligth Checks**

`kubeadm` 首先要做的是一系列的检查工作，以确定这台机器可以用来部署 `Kubernetes`。这一步检查，我们称为“`Preflight Checks`”，它可以为你省掉很多后续的麻烦。

`kubernetes\cmd\kubeadm\app\cmd\phases\init\preflight.go`

包括 `RunInitNodeChecks` 和 `RunPullImagesCheck`

```go
// runPreflight executes preflight checks logic.
func runPreflight(c workflow.RunData) error {
	data, ok := c.(InitData)
	if !ok {
		return errors.New("preflight phase invoked with an invalid data struct")
	}

	fmt.Println("[preflight] Running pre-flight checks")
	if err := preflight.RunInitNodeChecks(utilsexec.New(), data.Cfg(), data.IgnorePreflightErrors(), false, false); err != nil {
		return err
	}

	if !data.DryRun() {
		fmt.Println("[preflight] Pulling images required for setting up a Kubernetes cluster")
		fmt.Println("[preflight] This might take a minute or two, depending on the speed of your internet connection")
		fmt.Println("[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'")
		if err := preflight.RunPullImagesCheck(utilsexec.New(), data.Cfg(), data.IgnorePreflightErrors()); err != nil {
			return err
		}
	} else {
		fmt.Println("[preflight] Would pull the required images (like 'kubeadm config images pull')")
	}

	return nil
}
```

主要看下 `RunInitNodeChecks` 包括哪些检查

```go
// RunInitNodeChecks executes all individual, applicable to control-plane node checks.
// The boolean flag 'isSecondaryControlPlane' controls whether we are running checks in a --join-control-plane scenario.
// The boolean flag 'downloadCerts' controls whether we should skip checks on certificates because we are downloading them.
// If the flag is set to true we should skip checks already executed by RunJoinNodeChecks.
func RunInitNodeChecks(execer utilsexec.Interface, cfg *kubeadmapi.InitConfiguration, ignorePreflightErrors sets.String, isSecondaryControlPlane bool, downloadCerts bool) error {
	if !isSecondaryControlPlane {
		// First, check if we're root separately from the other preflight checks and fail fast
		if err := RunRootCheckOnly(ignorePreflightErrors); err != nil {
			return err
		}
	}

	manifestsDir := filepath.Join(kubeadmconstants.KubernetesDir, kubeadmconstants.ManifestsSubDirName)
	checks := []Checker{
    // 检查该节点的CPU是否大于等于2核
		NumCPUCheck{NumCPU: kubeadmconstants.ControlPlaneNumCPU},
		// 比较kubeadm 版本与待安装kuberntes版本，kubeadm 版本尽量与kubernetes版本一致
		KubernetesVersionCheck{KubernetesVersion: cfg.KubernetesVersion, KubeadmVersion: kubeadmversion.Get().GitVersion},
    // 检查防火墙是否关闭
		FirewalldCheck{ports: []int{int(cfg.LocalAPIEndpoint.BindPort), kubeadmconstants.KubeletPort}},
    // 检查apiserver 默认端口6443是否占用
		PortOpenCheck{port: int(cfg.LocalAPIEndpoint.BindPort)},
    // 检查SchedulerPort默认端口10259是否占用
		PortOpenCheck{port: kubeadmconstants.KubeSchedulerPort},
    // 检查ControllerManager10257是否占用
		PortOpenCheck{port: kubeadmconstants.KubeControllerManagerPort},
    // 检查/etc/kubernetes/manifest/kube-apiserver.yaml 是否存在
		FileAvailableCheck{Path: kubeadmconstants.GetStaticPodFilepath(kubeadmconstants.KubeAPIServer, manifestsDir)},
    // 检查/etc/kubernetes/manifest/kube-controller-manager.yaml 是否存在
		FileAvailableCheck{Path: kubeadmconstants.GetStaticPodFilepath(kubeadmconstants.KubeControllerManager, manifestsDir)},
    // 检查/etc/kubernetes/manifest/kube-scheduler.yaml 是否存在
		FileAvailableCheck{Path: kubeadmconstants.GetStaticPodFilepath(kubeadmconstants.KubeScheduler, manifestsDir)},
    // 检查/etc/kubernetes/manifest/etcd.yaml 是否存在
		FileAvailableCheck{Path: kubeadmconstants.GetStaticPodFilepath(kubeadmconstants.Etcd, manifestsDir)},
    // 检查api-server ip 是否可用
		HTTPProxyCheck{Proto: "https", Host: cfg.LocalAPIEndpoint.AdvertiseAddress},
	}
	cidrs := strings.Split(cfg.Networking.ServiceSubnet, ",")
	for _, cidr := range cidrs {
    // 检查serviceCidr 网段是否可用
		checks = append(checks, HTTPProxyCIDRCheck{Proto: "https", CIDR: cidr})
	}
	cidrs = strings.Split(cfg.Networking.PodSubnet, ",")
	for _, cidr := range cidrs {
    // 检查podCidr网段是否可用
		checks = append(checks, HTTPProxyCIDRCheck{Proto: "https", CIDR: cidr})
	}

	if !isSecondaryControlPlane {
    // 检查一些系统文件是否存在，iptables,mount等
		checks = addCommonChecks(execer, cfg.KubernetesVersion, &cfg.NodeRegistration, checks)

		// Check if Bridge-netfilter and IPv6 relevant flags are set
		if ip := net.ParseIP(cfg.LocalAPIEndpoint.AdvertiseAddress); ip != nil {
			if utilsnet.IsIPv6(ip) {
				checks = append(checks,
					FileContentCheck{Path: bridgenf6, Content: []byte{'1'}},
					FileContentCheck{Path: ipv6DefaultForwarding, Content: []byte{'1'}},
				)
			}
		}

		// if using an external etcd
		if cfg.Etcd.External != nil {
			// 检查etcd version
			checks = append(checks, ExternalEtcdVersionCheck{Etcd: cfg.Etcd})
		}
	}

	if cfg.Etcd.Local != nil {
		// 如果使用kubeadm 部署etcd，检查etcd端口，即/var/lib/etcd目录
		checks = append(checks,
			PortOpenCheck{port: kubeadmconstants.EtcdListenClientPort},
			PortOpenCheck{port: kubeadmconstants.EtcdListenPeerPort},
			DirAvailableCheck{Path: cfg.Etcd.Local.DataDir},
		)
	}

	if cfg.Etcd.External != nil && !(isSecondaryControlPlane && downloadCerts) {
		// Only check etcd certificates when using an external etcd and not joining with automatic download of certs
		if cfg.Etcd.External.CAFile != "" {
			checks = append(checks, FileExistingCheck{Path: cfg.Etcd.External.CAFile, Label: "ExternalEtcdClientCertificates"})
		}
		if cfg.Etcd.External.CertFile != "" {
			checks = append(checks, FileExistingCheck{Path: cfg.Etcd.External.CertFile, Label: "ExternalEtcdClientCertificates"})
		}
		if cfg.Etcd.External.KeyFile != "" {
			checks = append(checks, FileExistingCheck{Path: cfg.Etcd.External.KeyFile, Label: "ExternalEtcdClientCertificates"})
		}
	}

	return RunChecks(checks, os.Stderr, ignorePreflightErrors)
}
```

### Certs

这个步骤就是创建 k8s 集群组件相互访问的 TLS 证书，生成的证书结构如下：

```bash
$ /etc/kubernetes# tree pki/
pki/
├── apiserver.crt
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
├── apiserver.key
├── apiserver-kubelet-client.crt
├── apiserver-kubelet-client.key
├── ca.crt
├── ca.key
├── etcd
│   ├── ca.crt
│   ├── ca.key
│   ├── healthcheck-client.crt
│   ├── healthcheck-client.key
│   ├── peer.crt
│   ├── peer.key
│   ├── server.crt
│   └── server.key
├── front-proxy-ca.crt
├── front-proxy-ca.key
├── front-proxy-client.crt
├── front-proxy-client.key
├── sa.key
└── sa.pub
```

可以看到，使用 `kubeadm` 部署的 `kubernetes` 集群 会自签三套CA。

`ca.crt` 主要给`apiserver，scheduler，controller-manager,kubelet`签发证书，`kubelet` 客户端证书是 `controller-manager` 自动批准签发的，`kubelet-server` 证书因为安全问题，不会签发，`kubelet` 启动会自动生成。

`etcd/ca.crt` 主要给 `etcd server/client/peer`  签发证书

`front-proxy-ca.crt` 主要给扩展 `api-server` 签发证书

`kubeadm` 用于生成各个证书的重要代码段，他把各个证书的的生成封装成一个个的对象。然后放到我们subPhases中，上层的逻辑会挨个调用对象中的run函数。

```go
// newCertSubPhases returns sub phases for certs phase
func newCertSubPhases() []workflow.Phase {
	subPhases := []workflow.Phase{}

	// All subphase
	allPhase := workflow.Phase{
		Name:           "all",
		Short:          "Generate all certificates",
		InheritFlags:   getCertPhaseFlags("all"),
		RunAllSiblings: true,
	}

	subPhases = append(subPhases, allPhase)

	// This loop assumes that GetDefaultCertList() always returns a list of
	// certificate that is preceded by the CAs that sign them.
	var lastCACert *certsphase.KubeadmCert
	for _, cert := range certsphase.GetDefaultCertList() {
		var phase workflow.Phase
		if cert.CAName == "" {
			phase = newCertSubPhase(cert, runCAPhase(cert))
			lastCACert = cert
		} else {
			phase = newCertSubPhase(cert, runCertPhase(cert, lastCACert))
			phase.LocalFlags = localFlags()
		}
		subPhases = append(subPhases, phase)
	}

	// SA creates the private/public key pair, which doesn't use x509 at all
	saPhase := workflow.Phase{
		Name:         "sa",
		Short:        "Generate a private key for signing service account tokens along with its public key",
		Long:         saKeyLongDesc,
		Run:          runCertsSa,
		InheritFlags: []string{options.CertificatesDir},
	}

	subPhases = append(subPhases, saPhase)

	return subPhases
}
```

```go
// 获取待生成证书列表
func GetDefaultCertList() Certificates {
	return Certificates{
		&KubeadmCertRootCA, // 根证书
		&KubeadmCertAPIServer, // apiserver 服务端证书
		&KubeadmCertKubeletClient,  // apiserver 客户端证书，用于访问kubelet
		// Front Proxy certs
		&KubeadmCertFrontProxyCA, // 扩展apiserver 根证书
		&KubeadmCertFrontProxyClient, // 扩展apiserver client 证书，用于访问apiserver
		// etcd certs
		&KubeadmCertEtcdCA, // etcd 根证书
		&KubeadmCertEtcdServer, // etcd 服务端证书
		&KubeadmCertEtcdPeer, // etcd 邻居证书
		&KubeadmCertEtcdHealthcheck, // etcd 健康检查证书
		&KubeadmCertEtcdAPIClient, //  etcd api client 证书，用于apiserver 访问etcd
	}
}
```

### KubeConfig

这步生成 `kubeconfig` 文件，用于 `kubectl，controller-manager，scheduler` 访问 `apiserver`。

```go
// NewKubeConfigPhase creates a kubeadm workflow phase that creates all kubeconfig files necessary to establish the control plane and the admin kubeconfig file.
func NewKubeConfigPhase() workflow.Phase {
	return workflow.Phase{
		Name:  "kubeconfig",
		Short: "Generate all kubeconfig files necessary to establish the control plane and the admin kubeconfig file",
		Long:  cmdutil.MacroCommandLongDescription,
		Phases: []workflow.Phase{
			{
				Name:           "all",
				Short:          "Generate all kubeconfig files",
				InheritFlags:   getKubeConfigPhaseFlags("all"),
				RunAllSiblings: true,
			},
			// admin.conf 用于kubectl 
			NewKubeConfigFilePhase(kubeadmconstants.AdminKubeConfigFileName),
      // kubelet.conf 用于kubelet 访问api-server, 此时生成kubelet.conf 只是用于第一次部署，部署完成会向apiserver 申请csr重新生成正式文件。
			NewKubeConfigFilePhase(kubeadmconstants.KubeletKubeConfigFileName),
      // controller-manager.conf 用于controller-manager 访问api-server
			NewKubeConfigFilePhase(kubeadmconstants.ControllerManagerKubeConfigFileName),
      // scheduler.conf 用于scheduler 访问 api-server
			NewKubeConfigFilePhase(kubeadmconstants.SchedulerKubeConfigFileName),
		},
		Run: runKubeConfig,
	}
}
```

这些文件如果已经存在且有效，`kubeadm` 会跳过这步。

```go
// createKubeConfigFileIfNotExists saves the KubeConfig object into a file if there isn't any file at the given path.
// If there already is a kubeconfig file at the given path; kubeadm tries to load it and check if the values in the
// existing and the expected config equals. If they do; kubeadm will just skip writing the file as it's up-to-date,
// but if a file exists but has old content or isn't a kubeconfig file, this function returns an error.
func createKubeConfigFileIfNotExists(outDir, filename string, config *clientcmdapi.Config) error {
	kubeConfigFilePath := filepath.Join(outDir, filename)

	err := validateKubeConfig(outDir, filename, config)
	if err != nil {
		// Check if the file exist, and if it doesn't, just write it to disk
		if !os.IsNotExist(err) {
			return err
		}
		fmt.Printf("[kubeconfig] Writing %q kubeconfig file\n", filename)
		err = kubeconfigutil.WriteToDisk(kubeConfigFilePath, config)
		if err != nil {
			return errors.Wrapf(err, "failed to save kubeconfig file %q on disk", kubeConfigFilePath)
		}
		return nil
	}
	// kubeadm doesn't validate the existing kubeconfig file more than this (kubeadm trusts the client certs to be valid)
	// Basically, if we find a kubeconfig file with the same path; the same CA cert and the same server URL;
	// kubeadm thinks those files are equal and doesn't bother writing a new file
	fmt.Printf("[kubeconfig] Using existing kubeconfig file: %q\n", kubeConfigFilePath)

	return nil

```

### KubeletStart

创建 `kubelet` 启动时所需的配置文件

- 创建 `kubeadm-flags.env`
- 创建 `config.yaml`，这个`config.yaml`里的内容都是从`kubeadm-config.yaml`(kubeadm init 的文件)里的 `KubeletConfiguration` 直接拷贝过来。
- 启动 `kubelet`

那么会有疑问，`kubelet` 通过`systemd` 管理，不需要创建 `service` 文件吗？前文 [说明](https://www.notion.so/kubeadm-kubeadm-init-16d3569ee30942848cbd9b6402f2d30f) 已经说明

在使用 `yum` 安装`kubeadm` 时，`kubelet.service` 以及`kubelet.service.d/10-kubeadm.conf` 会自动安装。

```go
// runKubeletStart executes kubelet start logic.
func runKubeletStart(c workflow.RunData) error {
	data, ok := c.(InitData)
	if !ok {
		return errors.New("kubelet-start phase invoked with an invalid data struct")
	}

	// First off, configure the kubelet. In this short timeframe, kubeadm is trying to stop/restart the kubelet
	// Try to stop the kubelet service so no race conditions occur when configuring it
	if !data.DryRun() {
		klog.V(1).Infoln("Stopping the kubelet")
		kubeletphase.TryStopKubelet()
	}

	// 创建 kubeadm-flags.env
	if err := kubeletphase.WriteKubeletDynamicEnvFile(&data.Cfg().ClusterConfiguration, &data.Cfg().NodeRegistration, false, data.KubeletDir()); err != nil {
		return errors.Wrap(err, "error writing a dynamic environment file for the kubelet")
	}
  // 从 kubeadm-config.yaml 获取 kubelet 的配置
	kubeletCfg, ok := data.Cfg().ComponentConfigs[componentconfigs.KubeletGroup]
	if !ok {
		return errors.New("no kubelet component config found in the active component config set")
	}

	// 将kubelet-config 写入 config.yaml
	if err := kubeletphase.WriteConfigToDisk(kubeletCfg, data.KubeletDir()); err != nil {
		return errors.Wrap(err, "error writing kubelet configuration to disk")
	}

	// Try to start the kubelet service in case it's inactive
	if !data.DryRun() {
		fmt.Println("[kubelet-start] Starting the kubelet")
		kubeletphase.TryStartKubelet()
	}

	return nil
}
```

### ControlPlane

这步生成`master` 节点的`manifest` 文件，`kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml 。`

```go
// NewControlPlanePhase creates a kubeadm workflow phase that implements bootstrapping the control plane.
func NewControlPlanePhase() workflow.Phase {
	phase := workflow.Phase{
		Name:  "control-plane",
		Short: "Generate all static Pod manifest files necessary to establish the control plane",
		Long:  cmdutil.MacroCommandLongDescription,
		Phases: []workflow.Phase{
			{
				Name:           "all",
				Short:          "Generate all static Pod manifest files",
				InheritFlags:   getControlPlanePhaseFlags("all"),
				Example:        controlPlaneExample,
				RunAllSiblings: true,
			},
      // 生成 kube-apiserver.yaml
			newControlPlaneSubphase(kubeadmconstants.KubeAPIServer),
      // 生成 kube-controller-manager.yaml
			newControlPlaneSubphase(kubeadmconstants.KubeControllerManager),
      // 生成 kube-scheduler.yaml
			newControlPlaneSubphase(kubeadmconstants.KubeScheduler),
		},
		Run: runControlPlanePhase,
	}
	return phase
}
```

### Etcd

这步生成 `etcd manifest, etcd.yaml`

```go
func runEtcdPhaseLocal() func(c workflow.RunData) error {
	return func(c workflow.RunData) error {
		data, ok := c.(InitData)
		if !ok {
			return errors.New("etcd phase invoked with an invalid data struct")
		}
		cfg := data.Cfg()

		// 当 etcd 为local 安装时，才会创建 manifest
		if cfg.Etcd.External == nil {
			// creates target folder if doesn't exist already
			if !data.DryRun() {
				if err := os.MkdirAll(cfg.Etcd.Local.DataDir, 0700); err != nil {
					return errors.Wrapf(err, "failed to create etcd directory %q", cfg.Etcd.Local.DataDir)
				}
			} else {
				fmt.Printf("[dryrun] Would ensure that %q directory is present\n", cfg.Etcd.Local.DataDir)
			}
			fmt.Printf("[etcd] Creating static Pod manifest for local etcd in %q\n", data.ManifestDir())
			if err := etcdphase.CreateLocalEtcdStaticPodManifestFile(data.ManifestDir(), data.KustomizeDir(), cfg.NodeRegistration.Name, &cfg.ClusterConfiguration, &cfg.LocalAPIEndpoint); err != nil {
				return errors.Wrap(err, "error creating local etcd static pod manifest file")
			}
		} else {
			klog.V(1).Infoln("[etcd] External etcd mode. Skipping the creation of a manifest for local etcd")
		}
		return nil
	}
}
```

### WaitControlPlane

等待 `kubelet` 启动成功，超时时间为40s。

`kubelet` 启动成功后，`kubelet` 监听到 `manifest` 文件，就会自动拉起 `kube-apiserver、kube-controller-manager、kube-scheduler、etcd`。

```go
// WaitForKubeletAndFunc waits primarily for the function f to execute, even though it might take some time. If that takes a long time, and the kubelet
// /healthz continuously are unhealthy, kubeadm will error out after a period of exponential backoff
func (w *KubeWaiter) WaitForKubeletAndFunc(f func() error) error {
	errorChan := make(chan error, 1)

	go func(errC chan error, waiter Waiter) {
		if err := waiter.WaitForHealthyKubelet(40*time.Second, fmt.Sprintf("http://localhost:%d/healthz", kubeadmconstants.KubeletHealthzPort)); err != nil {
			errC <- err
		}
	}(errorChan, w)

	go func(errC chan error, waiter Waiter) {
		// This main goroutine sends whatever the f function returns (error or not) to the channel
		// This in order to continue on success (nil error), or just fail if the function returns an error
		errC <- f()
	}(errorChan, w)

	// This call is blocking until one of the goroutines sends to errorChan
	return <-errorChan
}
```

### UploadConfig

将 `kubeadm-config.yaml` 和 `kubelet-config.yaml` 作为 `data` 创建为 `configmap`，并创建可以获取该`configmap`的权限， 供`kubeadm join` 使用，这样其他节点只需从`configmap` load 配置即可。

```go
// NewUploadConfigPhase returns the phase to uploadConfig
func NewUploadConfigPhase() workflow.Phase {
	return workflow.Phase{
		Name:    "upload-config",
		Aliases: []string{"uploadconfig"},
		Short:   "Upload the kubeadm and kubelet configuration to a ConfigMap",
		Long:    cmdutil.MacroCommandLongDescription,
		Phases: []workflow.Phase{
			{
				Name:           "all",
				Short:          "Upload all configuration to a config map",
				RunAllSiblings: true,
				InheritFlags:   getUploadConfigPhaseFlags(),
			},
			{
				Name:         "kubeadm",
				Short:        "Upload the kubeadm ClusterConfiguration to a ConfigMap",
				Long:         uploadKubeadmConfigLongDesc,
				Example:      uploadKubeadmConfigExample,
        // 创建 kubeadm-config configMap
				Run:          runUploadKubeadmConfig,
				InheritFlags: getUploadConfigPhaseFlags(),
			},
			{
				Name:         "kubelet",
				Short:        "Upload the kubelet component config to a ConfigMap",
				Long:         uploadKubeletConfigLongDesc,
				Example:      uploadKubeletConfigExample,
        // 创建 kubelet-config configMap
				Run:          runUploadKubeletConfig,
				InheritFlags: getUploadConfigPhaseFlags(),
			},
		},
	}
}
```

### UploadCerts

这一步是用户可选配置，通过 `kubeadm init ——upload-cert`，将之前自签的**CA**作为`data` 创建`secret`，用于kubeadm join 使用，这样其他节点从`secret` load CA ，然后只用CA签发相应证书。

```go
func runUploadCerts(c workflow.RunData) error {
	data, ok := c.(InitData)
	if !ok {
		return errors.New("upload-certs phase invoked with an invalid data struct")
	}
  // 如果 kubeadm init 没有传递 --upload-cert 参数，则直接返回
	if !data.UploadCerts() {
		fmt.Printf("[upload-certs] Skipping phase. Please see --%s\n", options.UploadCerts)
		return nil
	}
	client, err := data.Client()
	if err != nil {
		return err
	}
  // 如果 kubeadm init 没有传递 certificate 参数，则会自动创建
	if len(data.CertificateKey()) == 0 {
		certificateKey, err := copycerts.CreateCertificateKey()
		if err != nil {
			return err
		}
		data.SetCertificateKey(certificateKey)
	}
  // 创建 cert secret
	if err := copycerts.UploadCerts(client, data.Cfg(), data.CertificateKey()); err != nil {
		return errors.Wrap(err, "error uploading certs")
	}
	if !data.SkipCertificateKeyPrint() {
		fmt.Printf("[upload-certs] Using certificate key:\n%s\n", data.CertificateKey())
	}
	return nil
}

//UploadCerts save certs needs to join a new control-plane on kubeadm-certs sercret.
func UploadCerts(client clientset.Interface, cfg *kubeadmapi.InitConfiguration, key string) error {
	fmt.Printf("[upload-certs] Storing the certificates in Secret %q in the %q Namespace\n", kubeadmconstants.KubeadmCertsSecret, metav1.NamespaceSystem)
	decodedKey, err := hex.DecodeString(key)
	if err != nil {
		return errors.Wrap(err, "error decoding certificate key")
	}
  // 创建 token 用于管理kubeadm cert，且会创建secret
	tokenID, err := createShortLivedBootstrapToken(client)
	if err != nil {
		return err
	}
  // 从本地证书目录加载证书内容
	secretData, err := getDataFromDisk(cfg, decodedKey)
	if err != nil {
		return err
	}
  // 将证书的secret 设置托管与 token 的secret ，使得声明周期一致
	ref, err := getSecretOwnerRef(client, tokenID)
	if err != nil {
		return err
	}
  // 创建 cert secret
	err = apiclient.CreateOrUpdateSecret(client, &v1.Secret{
		ObjectMeta: metav1.ObjectMeta{
			Name:            kubeadmconstants.KubeadmCertsSecret,
			Namespace:       metav1.NamespaceSystem,
			OwnerReferences: ref,
		},
		Data: secretData,
	})
	if err != nil {
		return err
	}
 
	return createRBAC(client)
}
```

### MarkControl

将该节点打上污点，通常`master`节点都会打上污点，不作为计算节点使用。

```go
// MarkControlPlane taints the control-plane and sets the control-plane label
func MarkControlPlane(client clientset.Interface, controlPlaneName string, taints []v1.Taint) error {

	fmt.Printf("[mark-control-plane] Marking the node %s as control-plane by adding the label \"%s=''\"\n", controlPlaneName, constants.LabelNodeRoleMaster)

	if len(taints) > 0 {
		taintStrs := []string{}
		for _, taint := range taints {
			taintStrs = append(taintStrs, taint.ToString())
		}
		fmt.Printf("[mark-control-plane] Marking the node %s as control-plane by adding the taints %v\n", controlPlaneName, taintStrs)
	}

	return apiclient.PatchNode(client, controlPlaneName, func(n *v1.Node) {
		markControlPlaneNode(n, taints)
	})
}

func taintExists(taint v1.Taint, taints []v1.Taint) bool {
	for _, t := range taints {
		if t == taint {
			return true
		}
	}

	return false
}
```

### BootstrapToken

创建`token`，用于其他节点加入集群时临时访问`apiserver`获取证书

```go
func runBootstrapToken(c workflow.RunData) error {
	data, ok := c.(InitData)
	if !ok {
		return errors.New("bootstrap-token phase invoked with an invalid data struct")
	}

	client, err := data.Client()
	if err != nil {
		return err
	}

	if !data.SkipTokenPrint() {
		tokens := data.Tokens()
		if len(tokens) == 1 {
			fmt.Printf("[bootstrap-token] Using token: %s\n", tokens[0])
		} else if len(tokens) > 1 {
			fmt.Printf("[bootstrap-token] Using tokens: %v\n", tokens)
		}
	}

	fmt.Println("[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles")
	// Create the default node bootstrap token
	if err := nodebootstraptokenphase.UpdateOrCreateTokens(client, false, data.Cfg().BootstrapTokens); err != nil {
		return errors.Wrap(err, "error updating or creating token")
	}
	// Create RBAC rules that makes the bootstrap tokens able to get nodes
	if err := nodebootstraptokenphase.AllowBoostrapTokensToGetNodes(client); err != nil {
		return errors.Wrap(err, "error allowing bootstrap tokens to get Nodes")
	}
	// Create RBAC rules that makes the bootstrap tokens able to post CSRs
	if err := nodebootstraptokenphase.AllowBootstrapTokensToPostCSRs(client); err != nil {
		return errors.Wrap(err, "error allowing bootstrap tokens to post CSRs")
	}
	// Create RBAC rules that makes the bootstrap tokens able to get their CSRs approved automatically
	if err := nodebootstraptokenphase.AutoApproveNodeBootstrapTokens(client); err != nil {
		return errors.Wrap(err, "error auto-approving node bootstrap tokens")
	}

	// Create/update RBAC rules that makes the nodes to rotate certificates and get their CSRs approved automatically
	if err := nodebootstraptokenphase.AutoApproveNodeCertificateRotation(client); err != nil {
		return err
	}

	// Create the cluster-info ConfigMap with the associated RBAC rules
	if err := clusterinfophase.CreateBootstrapConfigMapIfNotExists(client, data.KubeConfigPath()); err != nil {
		return errors.Wrap(err, "error creating bootstrap ConfigMap")
	}
	if err := clusterinfophase.CreateClusterInfoRBACRules(client); err != nil {
		return errors.Wrap(err, "error creating clusterinfo RBAC rules")
	}
	return nil
}
```

### KubeletFinalize

更换`kubelet.conf`，之前`kubelet.conf` 里的客户端证书是`kubeadm` 自动生成的，如果`kubelet` 开启了证书轮换，那么只要启动了`kubelet`，就会去和`apiserver` 申请证书，`controller-manager`会自动批准证书

```go
// runKubeletFinalizeCertRotation detects if the kubelet certificate rotation is enabled
// and updates the kubelet.conf file to point to a rotatable certificate and key for the
// Node user.
func runKubeletFinalizeCertRotation(c workflow.RunData) error {
	data, ok := c.(InitData)
	if !ok {
		return errors.New("kubelet-finalize phase invoked with an invalid data struct")
	}

	// Check if the user has added the kubelet --cert-dir flag.
	// If yes, use that path, else use the kubeadm provided value.
	cfg := data.Cfg()
	pkiPath := filepath.Join(data.KubeletDir(), "pki")
	val, ok := cfg.NodeRegistration.KubeletExtraArgs["cert-dir"]
	if ok {
		pkiPath = val
	}

	// Check for the existence of the kubelet-client-current.pem file in the kubelet certificate directory.
	rotate := false
	pemPath := filepath.Join(pkiPath, "kubelet-client-current.pem")
	if _, err := os.Stat(pemPath); err == nil {
		klog.V(1).Infof("[kubelet-finalize] Assuming that kubelet client certificate rotation is enabled: found %q", pemPath)
		rotate = true
	} else {
		klog.V(1).Infof("[kubelet-finalize] Assuming that kubelet client certificate rotation is disabled: %v", err)
	}

	// Exit early if rotation is disabled.
	if !rotate {
		return nil
	}

	kubeconfigPath := filepath.Join(kubeadmconstants.KubernetesDir, kubeadmconstants.KubeletKubeConfigFileName)
	fmt.Printf("[kubelet-finalize] Updating %q to point to a rotatable kubelet client certificate and key\n", kubeconfigPath)

	// Exit early if dry-running is enabled.
	if data.DryRun() {
		return nil
	}

	// 从目录里加载 kubelet.conf，此时的 kubelet.conf 是已经自动轮换的过的.
	kubeconfig, err := clientcmd.LoadFromFile(kubeconfigPath)
	if err != nil {
		return errors.Wrapf(err, "could not load %q", kubeconfigPath)
	}

	//通过 userName 验证该文件是否有效，防止被破坏
	userName := fmt.Sprintf("%s%s", kubeadmconstants.NodesUserPrefix, cfg.NodeRegistration.Name)
	info, ok := kubeconfig.AuthInfos[userName]
	if !ok {
		return errors.Errorf("the file %q does not contain authentication for user %q", kubeconfigPath, cfg.NodeRegistration.Name)
	}

	// Update the client certificate and key of the node authorizer to point to the PEM symbolic link.
	info.ClientKeyData = []byte{}
	info.ClientCertificateData = []byte{}
	info.ClientKey = pemPath
	info.ClientCertificate = pemPath

	// Writes the kubeconfig back to disk.
	if err = clientcmd.WriteToFile(*kubeconfig, kubeconfigPath); err != nil {
		return errors.Wrapf(err, "failed to serialize %q", kubeconfigPath)
	}

	// Restart the kubelet.
	klog.V(1).Info("[kubelet-finalize] Restarting the kubelet to enable client certificate rotation")
	kubeletphase.TryRestartKubelet()

	return nil
}
```

### Addon

安装coredns ，kube-proxy

```go
// NewAddonPhase returns the addon Cobra command
func NewAddonPhase() workflow.Phase {
	return workflow.Phase{
		Name:  "addon",
		Short: "Install required addons for passing Conformance tests",
		Long:  cmdutil.MacroCommandLongDescription,
		Phases: []workflow.Phase{
			{
				Name:           "all",
				Short:          "Install all the addons",
				InheritFlags:   getAddonPhaseFlags("all"),
				RunAllSiblings: true,
			},
			{
				Name:         "coredns",
				Short:        "Install the CoreDNS addon to a Kubernetes cluster",
				Long:         coreDNSAddonLongDesc,
				InheritFlags: getAddonPhaseFlags("coredns"),
				Run:          runCoreDNSAddon,
			},
			{
				Name:         "kube-proxy",
				Short:        "Install the kube-proxy addon to a Kubernetes cluster",
				Long:         kubeProxyAddonLongDesc,
				InheritFlags: getAddonPhaseFlags("kube-proxy"),
				Run:          runKubeProxyAddon,
			},
		},
	}
}
```

## 引用

[kubeadm init 原理](https://v1-19.docs.kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/)

[使用 kubeadm 配置集群中的每个 kubelet](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/kubelet-integration/#the-kubelet-drop-in-file-for-systemd)

[kubeadm 实现细节](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/implementation-details/)