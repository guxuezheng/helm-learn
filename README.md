# helm学习笔记

# **Helm是什么**

Helm是kubernetes中的应用的包管理工具，可以理解为kubernetes的中的yum工具，帮助解决k8s产品管理复杂的缺陷。

helm基本架构如下:

![](/assets/import.png)

# **Helm安装**

官方文档

[https://docs.helm.sh/using\_helm/\#installing-helm](#installing-helm)



Helm架构分为helmclient和tillerserver

Helmclient是helm的客户端，拥有对服务端各类对象的管理能力，包括repository、chart、release等。

Tillerserver是helm的服务端，服务端是由客户端通过init命令自动生成，包含deplment和service。



Helm需要用到socat，所以需要提前在物理机上安装

**\[root@master\_1 helm\]\# yum install -y socat**

接下来安装helm客户端，官方文档中，helm客户端的安装策略有很多，最简洁的是通过安装脚本安装，首先下载安装脚本

**curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get &gt; get\_helm.sh**

运行安装脚本:

**chmod 700 get\_helm.sh**

**./get\_helm.sh**

之后脚本会下载二进制文件并安装到path路径下。

安装完成之后，可以通过如下命令查看版本

**\[root@master\_1 helm\]\# helm version**

**Client: &version.Version{SemVer:"v2.8.1", GitCommit:"6af75a8fd72e2aa18a2b278cfe5c7a1c5feca7f2", GitTreeState:"clean"}**

由于此时，还未安装服务端，信息只有客户端版本信息，

接下来有客户端安装服务端

**\[root@master\_1 helm\]\# helm init**

由于helm默认采用的是rbac的认证方式，需要服务helm服务端特殊的权限，所以我们可以通过创建特定的服务账号来解决这个问题。这里我们创建helm服务账号。

执行命令之后，helm自动完成对服务端的创建，此时集群中将有如下信息:

**\[root@master\_1 helm\]\# kubectl get deployment --namespace=kube-system \|grep tiller**

**tiller-deploy              1         1         1            1           36m**

**\[root@master\_1 helm\]\# kubectl get svc --namespace=kube-system \|grep tiller**

**tiller-deploy         ClusterIP   10.254.86.148    &lt;none&gt;        44134/TCP                     37m**

此时执行helm命令，将提示未认证错误，如

**\[root@master\_1 helm\]\# helm ls**

**Error: Unauthorized**

因为我们未对helm服务账号赋予权限，执行如下命令

**\[root@master\_1 helm\]\# cat helm-rbac.yaml**

**apiVersion: v1**

**kind: ServiceAccount**

**metadata:**

**  name: helm**

**  namespace: kube-system**

**---**

**apiVersion: rbac.authorization.k8s.io/v1beta1**

**kind: ClusterRoleBinding**

**metadata:**

**  name: helm**

**roleRef:**

**  apiGroup: rbac.authorization.k8s.io**

**  kind: ClusterRole**

**  name: cluster-admin**

**subjects:**

**  - kind: ServiceAccount**

**    name: helm**

**    namespace: kube-system**



**\[root@master\_1 helm\]\# kubectl create -f helm-rbac.yaml**

**serviceaccount "helm" created**

**clusterrolebinding "helm" created**

再查看helm运行情况

**\[root@master\_1 helm\]\# helm ls**

**\[root@master\_1 helm\]\# helm search**

**NAME                          CHART VERSIONAPP VERSION  DESCRIPTION                                       **

**stable/acs-engine-autoscaler  2.1.2        2.1.1        Scales worker nodes within agent pools            **

**stable/aerospike              0.1.7        v3.14.1.2    A Helm chart for Aerospike in Kubernetes          **

**stable/anchore-engine         0.1.3        0.1.6        Anchore container analysis and policy evaluatio...**

到此，helm已经安装完成了！

这里有个问题就是需要提前具备访问外网得能力，因为服务端的安装需要从谷歌仓库中下载镜像。也可以将镜像预先下载下来或通过国内云\(类似阿里\)的镜像，通过

helm init –i $xxxxxx:xxxx命令修改镜像名称。

# **Helm基础概念**

官方文档

[https://docs.helm.sh/using\_helm/\#using-helm](#using-helm)



## 1.Charts:

Chart理解为helm中的软件包，它包含在Kubernetes集群内部运行应用程序，工具或服务所需的所有资源定义。可以等价理解为yum中的rpm包。

## 2.Repository:

仓库是一个收集和共享charts的地方，我们在安装过程中，并未设置仓库地址，helm通过search所搜索到的所有内容将来自stable仓库，可以对helm进行配置，让它使用其他仓库。

如:

**\[root@master\_1 helm\]\# helm repo list**

**NAME  URL                                             **

**stablehttps://kubernetes-charts.storage.googleapis.com**

**localhttp://127.0.0.1:8879/charts  **

## 3.Release

一个在集群中运行的charts的实例，charts的运行时，在同一个集群中，一个charts可以被多次安装，比如一个mysql的charts，被安装两次，每次安装都会生成一个新的release，每个release都有一个独立的名称。







# **Helm使用**

## **charts的安装**

Helm安装成功之后，默认是可用stable的官方仓库，

可通过命令查看

**helm search**

支持过滤条件

**helm search mysql**

此处的过滤，并非对charts名称进行过滤，而是对charts内详细的各类描述信心进行过滤。

通过查询，我们安装tomcat

**helm install stable/tomcat**

这步骤将安装tomcat 的depl  service等一系列应用

当然，此时下载的，一定还是k8s官方的数据库，假如没有科学上网的话，很难下载下载，我们可以通过delete命令删除版本

**helm delete\[**release-name**\]**

删除之后，我们可以自定义charts信息，最简单的办法是得到config文件，编辑之后，使用我们编辑之后的文件进行启动

首先拿到原配置文件

helm inspect values \[charts-name\]

此时会打印一组数据，我们将数据复制到一个文件中config.yaml

如下:

![](file:///C:\Users\ASUS\AppData\Local\Temp\ksohtml11368\wps3.jpg)

此时我们就可以更改配置内容，从而实现定制化，此处我把tomcat的image改了私有仓库的地址。之后执行以更改后的配置文件执行charts

helm install -f config.yaml stable/tomcat

除了我们下载下来的方式，由于配置文件是yaml，可选择性的更改某些项配置内容。

通过**  —set**

如我们调整端口号:

**helm install stable/tomcat –setservers\[0\].port=1080**

更多调整配置的设置，查看官方文档

[https://docs.helm.sh/using\_helm/\#installing-helm](#installing-helm)



## **更新与回滚**

由于是入门操作，还是按照配置文件的方式，对charts和release进行更新，

首先修改对应的配置文件,修改配置文件后执行:

**helm upgrade -f panda.yaml aged-turkey stable/tomcat**

此时，可以通过命令查看release更新状态

**helm get values aged-turkey**

打印信息可以看到release已经被更新了。

回滚操作首先需要知道历史版本号

**helm history \[RELEASE\]**

如

**\[root@kube-0 tomcat\]\# helm history aged-turkey**

**REVISIONUPDATED                 STATUS    CHART       DESCRIPTION     **

**1       Mon Aug 20 17:06:56 2018SUPERSEDEDtomcat-0.1.0Install complete**

**2       Tue Aug 21 14:26:10 2018SUPERSEDEDtomcat-0.1.0Upgrade complete**

**3       Tue Aug 21 14:28:33 2018DEPLOYED  tomcat-0.1.0Upgrade complete**

此时可直接执行回滚操作，将release版本更新到之前版本



## **删除release**

**\[root@kube-0 ~\]\# helm delete aged-turkey**

**release "aged-turkey" deleted**

可以通过

helm list

查看所有的release，我们可以看到，此时已经将release删除，但并非是无处可寻了，我们可以通过 –deleted 或–all显示出这些已经被删除掉的release

**\[root@kube-0 ~\]\# helm list --deleted**

**NAME          REVISIONUPDATED                 STATUSCHART       APP VERSIONNAMESPACE**

**aged-turkey   5       Tue Aug 21 14:38:34 2018DELETEDtomcat-0.1.07          default  **

**quiet-anteater1       Mon Aug 20 16:49:06 2018DELETEDtomcat-0.1.07          default  **

此时我们知道release的版本名称和历史版本，可以通过回滚操作回滚到某个实现版本

回滚后，仍然将按照之前运行状态运行。



## **自定义charts**

参考了个技术大牛的博客

[http://www.sohu.com/a/216705143\_262549](http://www.sohu.com/a/216705143_262549)

谈到自定义charts，我们可以需要先明白helm到底是做什么的，在很多的应用场景中，我们有很复杂的部署过程，这样的部署过程可能需要很多的详细配置，比如启动redis集群和mysql集群，通过容器技术启动这样的集群还是需要一定的复杂度的，那么我们是不是可以通过一些手段，从而简化这种配置的过程呢，helm就是基于此场景应用而生的。

自定义charts的过程，简单说其实就是抽离了一些变量以供用户可以自定义，剩余大部分配置文件都预先定义好，同时charts一般都会整合depl、svc和ingress。

我们可以纯手写charts的配置，也可以通过helm为我们提供构建模板

因为开始不会写啊，只能先通过helm为我们提供的模板来创建:

**helm create demo1**

执行之后，树形结构如下:

**\[root@kube-0 deis-workflow\]\# tree demo1**

**demo1**

**├──charts**

**├──Chart.yaml**

**├──templates**

**│├──deployment.yaml**

**│├──\_helpers.tpl**

**│├──ingress.yaml**

**│├──NOTES.txt**

**│└──service.yaml**

**└──values.yaml**

暂时先不管都是干嘛的，一会再想，此时这个目录结构就是一个charts，我们可以通过package命令为charts打包

**helm package\[dir-name\]**

在打包之间，可以通过lint命令检查一下charts是否可用

**\[root@kube-0 deis-workflow\]\# helm lint demo1**

**==&gt; Linting demo1**

**\[INFO\] Chart.yaml: icon is recommended**

**1 chart\(s\) linted, no failures**

之后执行打包操作，将生成一个压缩包

此时，通过install命令就可以将我们自己的charts发布成一个release了，然后相关的depl，svc就都有了，因为我们没定制化，helm帮我们做了定制化，这里启动的是一个nginx镜像。

**\[root@kube-0 demo1\]\# helm ls**

**NAME          REVISIONUPDATED                 STATUS  CHART      APP VERSIONNAMESPACE**

**torrid-maltese1       Tue Aug 21 15:49:37 2018DEPLOYEDdemo1-0.1.01.0        default  **

可以看到，我们的helm跑起来啦。

说话的自定义和定制化呢，一脸懵逼。我们再回过头看一下charts的树形结构。

我们现在为相关的文件加上解释，如下:

**\[root@kube-0 demo1\]\# tree**

**.**

**├──charts**

**├──Chart.yaml   \#chart的源数据的相关记录版本和名称等**

**├──templates**

**│├──deployment.yaml\#depl的相关模板信息，内部存在许多变量**

**│├──\_helpers.tpl\#一脸懵逼**

**│├──ingress.yaml\#ingress的相关模板信息，内部存在许多变量**

**│├──NOTES.txt\#配置一些变量加载的流程逻辑和实现策略**

**│└──service.yaml\#svc的相关模板信息，内部存在许多变量**

**└──values.yaml\#变量存储位置，启动发布release时替换templates下的相关模板中的变量**

我们此时先观察一下deployment.yaml

![](file:///C:\Users\ASUS\AppData\Local\Temp\ksohtml11368\wps4.jpg)

可以看到大量的变量，这些变量都可以在values.yaml、chart.yaml或发布release时得到，诸如此，ingress.yaml和service.yaml相关配置也在其中可以找到，树形结构中有一个NOTES.txt，此处有一段代码是判断是否加载ingress的处理逻辑的，有兴趣可以去看一下。

此时我们明白了chart的作用是将部署复杂应用变得简单，我们在处理一类应用系统时，可以将这类应用系统统一打包为一个chart，抽离出一些可能会调整的变量，剩余的大部分全部设置为固定值，这样我们就不需要每次都重新配置如此复杂的应用系统了。









