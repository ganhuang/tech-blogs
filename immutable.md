# 使用 Packer、Ansible 和 Terraform 构建不可变的基础设施

在容器编排领域，Kubernetes 已成为事实上的标准。容器技术极大的推动了企业内部 Devops 运动的进程。容器镜像（Docker Image）是容器技术栈中最关键的创新之一。

容器镜像所具有的轻量性、便携性、分层机制、内核共享机制真正意义上实现了 “Build once, run anywhere"。这种不可变的基础设施（Immutable Infrastruture）高度保持了开发、测试和生产环境的一致性。因为镜像的易移植、易复制的特性，也给运维带来了很大的弹性和灵活性。

对于还无法容器化，只能部署在虚拟机里的传统应用，该如何构建不可变的的基础设施呢？

### 可变的服务器部署 vs 不可变的服务器部署

在可变的服务器部署模式中，首先我们通过 [Terraform](https://www.terraform.io/) 创建出所需的虚拟机以及其他基础设施资源，然后通过配置管理工具 [Ansible](https://www.ansible.com/) 对已经存在的服务器进行应用的配置和部署。

上述的部署过程中看似非常快速、简便，但是随着业务的需求增加，需要对服务器操作系统进行更新，或对部署的应用进行频繁的升级。这种情况下，“可变的服务器部署” 所带来的挑战和风险是我们无法提前预估的。

在真实的用户场景里，运行的应用程序与操作系统、第三方资源存在各种各样复杂的依赖。当给操作系统打补丁时，亦或是升级应用程序所依赖的一个软件包时，可能会出现应用程序无法正常运行、DNS 解析异常、网络不可达等。出错的原因是我们无法控制，也无法预测的。

即使应用程序更新成功了，一旦线上环境产生不可预知严重的 Bug，需要将应用程序回滚时，因为 “可变的服务器部署” 的不确定性，回滚的过程对于运维人员仍然是一项挑战。

当线上环境负载过高时，“可变的服务器部署” 模式往往也会显得不够高效。按照上述流程，需要创建新的虚拟机资源，再去运行配置管理工具去部署应用。

相对于可变的服务器部署模式，不可变的服务器部署模式要求运行的服务在部署之后，不能再对服务器做任何变更。如果需要对服务器系统更新、或对应用升级，需要重新创建新的服务器来替代旧的服务器。

### 基础设施即代码 (IAC)

基于 Packer、Ansible 和 Terraform 等开源工具构建出持续集成和持续部署的 Jenkins Pipeline：

![打包部署流程](immutable.png)

#### 应用代码打包

为了使部署更加灵活，基础设施及配置代码独立于应用本身代码，单独存储。

使用 [Jenkins Multibranch](https://jenkins.io/doc/book/pipeline/multibranch/), 应用代码仓库会基于分支自动触发 Build 并上传至中央存储仓库。

#### 虚拟机镜像打包 Packer

[Packer](https://www.packer.io/) 是一个优秀的，开源镜像打包工具。
Packer 的 builder 支持主流的公有云、私有云平台以及常见的虚拟化类型。

同时它支持的 provisioner 能覆盖主流的配置管理工具: Ansible, Puppet, Chef, Windows Shell, Linux Shell 等.

#### 配置管理及安全加密 Ansible

[Ansible](https://www.ansible.com/) 是一款简单的，易上手的开源配置管理工具。它能简化软件的安装部署，作为配置管理能提供灵活的模版渲染引擎以及[针对敏感信息的加密](https://docs.ansible.com/ansible/latest/user_guide/vault.html#)。

#### 基础设施的创建和编排 Terraform

[Terraform](https://www.terraform.io/) 作为开源的基础设施资源编排工具，能覆盖主流的云平台，非常适用于多云的环境。能提供灵活的部署选择，并能根据用户需求开发可插拔式的、自定义的 provider。

### 镜像部署过程中所面临的挑战

业务场景的不通，会要求部署方式的多样化，比如滚动部署、蓝绿部署等。

下面将介绍两种较典型的应用类型所面临的问题：

* 负载均衡器 (LB) + 应用服务器 (Web Server)
* 有状态的后端应用

***Note***: 主流的云厂商提供了类似动态虚拟机组的功能，来满足以上两种需求。本文主要介绍使用 Terraform 构建通用的解决方案。

#### 负载均衡器配置的平滑更新

在 LB + Web Server 这种业务场景下，为了尽量减少服务不可用的时间，制定了蓝绿部署的解决方案。

在资源池中，会存在蓝和绿两种虚拟机组。每次版本更新时，会选择**非**线上版本的一组虚拟机组做更新。

当**非**线上的版本更新完毕之后，会获取新创建的虚拟机 (VM) 的 IP 列表，将其动态更新至 LB 的后端。

在对 LB 进行更新时，定义该资源的 [lifecycle](https://www.terraform.io/docs/configuration/resources.html#lifecycle-lifecycle-customizations) 为 `create_before_destroy = true`。 这样每次更新时会先把新的后端虚拟机 IP 添加至 LB，待所有新虚拟机组的后端 IP 加入完毕之后，terraform 再去移除老的虚拟机 IP 组.

```
resource "cloud_load_balance" "lb" {
  # ...

  lifecycle {
    create_before_destroy = true
  }

}
```

#### 有状态应用的平滑升级

同样为了有状态的应用更平滑的更新。在老版本虚拟机销毁之前，需要发送一些个性化的指令，让应用程序能够优雅地退出。

除了对该虚拟机组资源的 lifecycle 指定 `create_before_destroy = true`, 还指定了一个 `local-exec` 的 provisioner 去优雅的停掉老虚拟机组里的应用。

***Note***: 在本例子中，脚本 `drain_nodes.sh` 会相对复杂，因为会并行创建多台虚拟机，所以需要加入类似锁的机制来避免竞争的情况发生。


```
resource "cloud_vm_instance" "instances" {
  count = "${var.instance_count}"

  # ...
  lifecycle {
    create_before_destroy = true
  }

  provisioner "local-exec" {
    command = "bash ${path.module}/scripts/drain_nodes.sh ${var.old_instance_list}
  }

}
```

***Note***: 由于 [terraform issue](https://github.com/hashicorp/terraform/issues/13549), 当指定了 `create_before_destroy = true` 时, 不能再使用 [Destroy-Time Provisioners](https://www.terraform.io/docs/provisioners/index.html#destroy-time-provisioners)。

#### 部署的可靠性和稳定性

为了提高部署的可靠性，在摧毁老的虚拟机组或者更新 LB 配置之前，需要确保新创建的虚拟机是健康可用的。为此从两个角度去优化：

* 为了尽早发现潜在的问题，在使用 Packer 打包镜像的时候，加入简单的健康检查机制，确保应用代码和配置是匹配的。
* 在新的虚拟机启动之后，加入自我健康检查的脚本：

```
resource "cloud_vm_instance" "instances" {

  # ...

  provisioner "local-exec" {
    command = "bash ${path.module}/scripts/health_check.sh ${self.ipv4_address}
  }

}
```

#### 镜像的打包效率

相对于可变服务器部署模式，由于打包虚拟机镜像过程较为耗时，在一定程度上会加长整个部署的时间。

因为镜像里包含了应用程序所需要的代码和配置，每一次的配置更新或者代码更新需要重新打包镜像。可以考虑把配置和代码从镜像中分离出来提高打包效率：

* 将镜像分层管理，分为基础镜像和应用镜像。在构建应用镜像不必再重新安装基础镜像中存在的基础软件包、配置，相对减少了构建时间。
* 将配置迁移至配置管理服务，应用程序启动时从该配置服务中动态获取配置信息
* 将配置和代码迁移至网络文件存储，虚拟机每次启动时挂载该网络文件存储去读取配置和代码

#### 镜像构建一次，多处部署

不同的环境 (Dev, QA, Prod), 会对应不同的配置文件。云环境中，支持给虚拟机传入 `user_metadata` 去区分不同的环境，由于镜像中包含所有环境的配置文件，可以通过传入的 `user_metadata` 去选择相应的配置文件启动应用程序。

```
resource "cloud_vm_instance" "instances" {

  # ...
  user_metadata = "dev"

}
```

#### 快速伸缩和回滚

从运维角度来看，可伸缩、可回滚性是平台维护中不可或缺的特性。

在 Terraform 中，我们可以通过简单的指定 `count` 数量来伸缩虚拟机数量：

```
resource "cloud_vm_instance" "instances" {

  count = "${var.instance_count}
  # ...

}
```

由于镜像包含应用程序所需要的所有配置和代码，虚拟机镜像的版本也就代表了应用程序的版本。回滚应用程序相当于指定虚拟机镜像的版本重新部署：

```
resource "cloud_vm_instance" "instances" {

  # ...
  image_version = "${var.image_version}"

}
```

### 总结

相对于 [AWS auto scaling group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)、[Google Autoscaling groups](https://cloud.google.com/compute/docs/autoscaler/) 和 [Azure VMSS](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview) 等公有云平台提供的原生服务，这种通用的解决方案显得有所欠缺。但在多云的环境、云平台提供的原生服务有所欠缺时，像这种基于 Terraform 本身构造的解决方案仍能体现其价值。在实际场景中用户可以灵活选择。

不可变服务器部署模式由于延长了从代码提交到收到部署反馈的时间，在一定程度上牺牲了开发体验。但权衡来看，它所带来的环境的一致性、运维的弹性，仍然是 Devops 运动中首选的解决方案。
