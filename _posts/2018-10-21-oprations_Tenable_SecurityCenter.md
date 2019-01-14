---
layout: post
title: 使用SecurityCenter实现企业安全闭环
date: 2018-10-21
tags: 安全工具
---

## 介绍

　　`Tenable SecurityCenter` (安全中心) 是一个集成了漏洞管理与分析的安全项目(这里简称 `SC` ), 说到这里 Tenable 的拳头产品 `Nessus` 漏洞扫描器作为安全行业顶尖的安全项目相信安全从业者并不陌生, `SC` 作为 `Nessus` 漏洞扫描引擎的分布式集成中心,集成了对Nessus漏洞扫描引擎的数据集成、数据展示、漏洞分析、授权、漏洞组件更新、安全基线、管理、格式化仪表板等持续化构建。

　　在本文中，我将介绍 `SecurityCenter` 对大规模数据集成中心的安全管理及安全运营, 也会介绍该项目实际的使用与维护中的一些缺陷, 用于给安全从业者做一个企业安全架构中漏洞管理平台的参考。                  


<div align="center">
	<img src="/images/posts/tenable/SecurityCenter.png" height="300" width="500">  
</div>


### 目录

* [SC安装](#sc-install)
* [Nessus漏洞扫描器引擎安装](#nessus-install)
* [SC关联并授权Nessus漏洞扫描器](#sc-association-nessus)
* [SC组织创建](#sc-organization-create)
* [SC数据仓库概念及区域性划分](#sc-repositories)
* [SC扫描空间概念及区域性划分](#sc-scanzone)
* [SC资产管理](#sc-asset-manage)
* [SC安全基线](#sc-sec-baseline)
* [SC漏洞扫描](#sc-vulnerability-scanning)
* [SC仪表板](#Where-to-go-from-here)
* [SC简单漏洞运营](#Where-to-go-from-here)


### <a name="sc-install"></a>SecurityCenter安装

　　`SecurityCenter` 的安装包需要在[Tenable官方网站](https://www.tenable.com/downloads)进行下载, 由于方便运维团队进行自动化管理, 本文Tenable的运行环境为 `Centos 7.x` 或 `Centos 6.x`, 所以下载的SC安装包为rpm包。Tenable 官方将 `SC` 定位成一个漏洞数据集成管理中心, 它本身不具备主动扫描漏洞的能力, 所以下载SC安装包的同时还需要下载Nessus的rpm安装包, SC与Nessus是收费版软件要想使用必须先购买license, 购买了SC的license之后就可以直接对Nessus进行授权管理, 一个SC的license可以授权管理多少的Nessus这里就不多谈, 小伙伴们可以自己去了解。

　　关于如何使用 `SC` 我这里只能讲解已经在使用或者已经测试过的一些功能, 还有部分功能如主机安全等并未购买相关服务及功能这里无法细说。 通过几个月对 `SC` 的使用, 发现这个产品的特点及局限性有了一个大致的了解, 现在让我们来安装它：

* **首先，需要在[Tenable官方](https://www.tenable.com/downloads)下载SC与Nessus的安装包**，由于我这里的网络环境复杂且需要安装管理的Nessus扫描器 (这里称为Nessus节点) 较多, 考虑到开启大量扫描时SC的性能分配给SC的配置为8core、32G、1TB (IDC规模更大的话可以增加配置) 作为管理节点运行, Nessus的数量可以按照业务与服务器体量的大小来进行安装配置。 由于SC只是通过Nessus进行漏洞扫描,使用SC调用Nessus扫描时Nessus本地不会保存扫描结果数据, 所以分配给Nessus的资源可以参考对单个节点Nessus使用情况来分配。

* **安装SC前先确认Centos系统是否是最简化安装的**, 最简化安装的 `Centos` 系统在安装  `SC` 之前需要安装前置依赖, 安装命令如下:
```shell
	yum -y install libxslt zip rsync unzip
	rpm -ivh SecurityCenter-5.7.1-el7.x86_64.rpm
```

* **安装完成后如何确认正确安装?**, 很简单, 只要看到如下图即安装成功。      

<div align="center">
	<img src="/images/posts/tenable/sc_install_success.png" height="300" width="500">  
</div>

* **Web管理界面**, 访问 `https://ip` 即可进入首次授权页面。

<div align="center">
	<img src="/images/posts/tenable/sc_accredit.png" height="300" width="500">  
</div>

* **授权激活事项**, 上传license激活SC, 激活SC的同时需要使用激活码激活Nessus扫描器引擎, 否则SC只会给Nessus进行授权激活无法更新推送插件及策略, 登陆Nessus会提示插件没有更新。

<div align="center">
	<img src="/images/posts/tenable/activation_nessus.png" height="300" width="500">  
</div>

### <a name="nessus-install"></a>Nessus漏洞扫描器引擎安装

　　`Nessus` 漏洞扫描器作为目前全球使用最广泛最多的漏洞扫描与分析软件, 它在系统漏洞扫描方面的能力毫无疑问是非常强悍的, 但在web应用方面的扫描能力上缺点也明显, 它无法同 `AWVS 10.x` 版本一样进行动态的web应用登陆扫描功能, 不然的话Nessus就更加完美了, 虽然不能进行`动态`的web应用登陆扫描但其对无登陆认证web页面扫描能力也不弱。

　　在本文中将使用 `Nessus` 与 `SC` 进行联动, 使该项目成为一个真正的综合型漏洞数据分析中心与安全运营管理平台, 现在让我们来安装它：

* **Nessus安装命令**, 将 `Nessus` 安装包上传到服务器, 这里的安装包依然是rpm包, 其他安装方式请查看官方安装文档, 安装命令如下:
```shell
   rpm -ivh rpm -ivh Nessus-8.0.0-es7.x86_64.rpm
```

<div align="center">
	<img src="/images/posts/tenable/nessus-install.png" height="300" width="500">
</div>

* **web管理界面**, 访问 `https://ip:8834` 进入 `Nessus` 初始化页面并设置登陆账号密码。

<div align="center">
	<img src="/images/posts/tenable/nessus_create_account.png" height="300" width="250">
</div>

* **选择扫描器类型**, 在选择扫描器类型选择 `Managed by SecurityCenter` 使用 `SC` 来管理并授权 `Nessus`, 之后下一步等待初始化安装。

<div align="center">
	<img src="/images/posts/tenable/nessus_select_scanner_type.png" height="300" width="250">
</div>

### <a name="sc-association-nessus"></a>SC关联并授权Nessus漏洞扫描器

　　安装完成 `SC` 与 `Nessus` 之后为了联动需要将 `Nessus` 关联配置到SC的Nessus管理中心去才可以实现分布式的漏洞扫描检测与漏洞信息侦察。

* **Step 1** 使用系统管理员账号登陆 `SC`, 点击 Resources->Nessus Scanners 在Nessus管理界面进行配置
<div align="center">
	<img src="/images/posts/tenable/sc-association-nessus-addNessus.png" height="" width="800">
</div>

* **Step 2** 填写Nessus相关信息, `Name` 可以按照相关的业务领域进行规划命名, `Host` 为安装的 `Nessus` 扫描器的IP地址, `Port` 如果在安装 `Nessus` 时没有变更的话默认即可, `Authentication` 填写的是 `Nessus` 账号密码, `Active Scans` 也就是 `Nessus` 活动范围的配置需要根据SC组织及扫描空间的划分来进行关联匹配。详情请查看<a name="sc-organization-create"> SC扫描空间概念及区域性划分
<div align="center">
	<img src="/images/posts/tenable/sc-association-nessus.png" height="" width="800">
</div>

* **Step 3** 配置完成后提交, `SC` 会自动进行关联授权以及插件更新并推送到 `Nessus`.
<div align="center">
	<img src="/images/posts/tenable/sc-association-nessus-complete.png" height="" width="800">
</div>

### <a name="sc-organization-create"></a>SC组织创建

　　`SC` 的组织个人理解为多个不同组件的集合体, 它的作用就是为了让 `SC` 所有组件更好的交叉配合使用。要想正常使用SC的所有功能必须要先创建组织.

* **Step 1** 使用系统管理员账号登陆 `SC`, 点击 Organizations->Add 添加组织.
<div align="center">
	<img src="/images/posts/tenable/sc-organizations-create.png" height="" width="800">
</div>

* **Step 2** `Name` 组织名称可按照实际的公司名或者团体名称进行命名, 包含地址、城市、邮件、电话等. `Scanning` 这里可以按照实际的业务规划来进行配置, 举个栗子: 假如xxx集团下属包含xxx个公司, 那么就可以使用该配置来对该下属公司的活动空间进行限制, 限制方式是以对该公司的网络结构以及规划来进行的. 这里为了举栗, 我选择 `Automatic Distribution Only` (自动分发), `Restricted Scan Ranges` 配置为 `0.0.0.0/0`, 其他配置项为了业务稳定可按需进行配置, 如ldap服务器等. `Analysis` 选项需要参考 **SC数据仓库概念及区域性划分** 来配置, 主要配置还是需要参考业务领域的规划及划分进行选择, 该选项可以使用领域划分或者功能性划分来进行选择配置, 需要对数据仓库以及扫描策略进行关联划分, 说简单点就是用于关联配置扫描策略的选项. `Custom Analysis Links` 与 `Vulnerability Weights` 可使用默认配置, 前者用于关联分析链接配置, 后者用于自定义漏洞权重. 配置完成提交即可.

<div align="center">
	<img src="/images/posts/tenable/sc-organizations-add.png" height="" width="800">
</div>

<div align="center">
	<img src="/images/posts/tenable/sc-organizations-add-2.png" height="" width="800">
</div>

### <a name="sc-repositories"></a>SC数据仓库概念及区域性划分

　　`SC` 的组织对企业资产进行了比较大范围的划分, 而数据仓库用于将这些资产进行更加精细化的划分梳理, 可以参考树形图, 它的主要作用是更加清晰明了对某个部门或者某个业务模块进行模块化配置, 当然如果一定要用它来配置更大的资产模块也是可行的, 可根据实际情况及项目规划来运维. 它还有一个更重要的作用是用来对资产进行重组划分, 之所以要进行这么精细划分的目的是因为可以有效防止企业网络规模过大且网络域、网段重叠的情况. 所以企业网络规划清晰明了且不具备重叠情况的话通常最多只需要创建2个数据仓库即可, 这里我将安全基线的检查与漏洞扫描进行区分创建了2个仓库, 按照业务领域增加了基地与总部的数据仓库(因为网络段重叠), 共计4个.
<div align="center">
	<img src="/images/posts/tenable/sc-repositories.png" height="" width="800">
</div>

* **Step 1** 点击Repositories-Add 增加数据仓库, 根据提示选择资产模式, 这里可以手动添加资产也可以按照资产模板进行导入, 这里选择添加IPV4仓库.
<div align="center">
	<img src="/images/posts/tenable/sc-repositories-add.png" height="" width="800">
</div>

* **Step 2** `Name` 的命名最好按照实际规划的业务领域进行命名, `Data` 为该xx部门或者业务进行活动范围限制这里也需要根据实际的网络划分来进行限制用于测试可以直接使用 `0.0.0.0/0`, `Access` 为仓库关联已经配置好的组织, `Advanced Settings` 这里选择默认, 如果有购买LCE服务可以填写, 配置完成后提交即可.
<div align="center">
	<img src="/images/posts/tenable/sc-repositories-add-2.png" height="" width="800">
</div>

### <a name="sc-scanzone"></a>SC扫描空间概念及区域性划分

　　`scanzone` 主要是为 `Nessus` 扫描器的活动范围进行限制, 能更加精细化的控制某个系统或某个ip地址的工作将会由哪个 `Nessus` 扫描器进行工作.

* **Step 1** 点击 `Resources->Scan Zones->Add` 进行添加扫描空间.
<div align="center">
	<img src="/images/posts/tenable/sc-scanzone-add.png" height="" width="800">
</div>

* **Step 2** `Name` 这里的名字可以按照实际的业务模块进行划分例如xx公司-xx领域或xx部门来进行命名, `Ranges` 为该业务部门或者领域设定活动范围以具体的网段划分来设置, `Scanners` 用于关联该领域或者业务部门的 `Nessus` 漏洞扫描器, 配置完成提交即可, 注意: 配置完成到这里就可以将已经安装并授权的 `Nessus` 的扫描空间进行关联了详情请查看<a name="nessus-install"> Nessus安装

### <a name="sc-asset-manage"></a>SC资产管理
　　整个漏洞中心 `SC` 的资产管理在整个项目中相当于上下文的作用, 对下给漏洞扫描引擎提供基础数据的支持, 对上给整个项目提供漏洞数据格式化展示、漏洞分析提供有效的数据, `SC` 的资产管理可以根据实际业务及IDC中心的规模进行动态管理. 在实际应用中比较常见的做法是直接通过 `CMDB` 与 `SC` 进行动态关联或直接按照网络规划的网段进行管理, 前者能有效的进行服务器登陆检测、安全配置检测, 后者可以全面探测资产存活、资产漏洞等. 注意: 为了更加顺畅并高效的使用 `SC` 这里将账号权限做了一个比较明确的定义及使用规范, `admin` 账号用于管理整个系统层面的功能及资源如 `Nessus` 扫描器的管理、业务领域的划分、扫描空间定义等, `admin` 账号的权限作为最高级别的父级权限, 使用 `admin` 账号创建一个二级权限用于 `漏洞扫描`、`漏洞数据分析`、`资产管理`、`漏洞仪表板`、`漏洞格式化报表` 等功能的账号, 到这里权限分级及分类管理就基本实现.

<div align="center">
	<img src="/images/posts/tenable/sc-asset-manage.png" height="" width="800">
</div>

* **Step 1** 使用二级权限账号登陆, 点击 `Assets->Add` 来增加资产. `Tenable` 提供了大量常见的资产模板, `Templates` 中的各个资产模板需要到 `Tenable` 进行查询使用, 这里我常用的是 `Custom` 列表中的资产模板 `Static IP List`、`DNS Name List`、`Combination`.
`Static IP List` 主要用于管理配置静态的IP资产, `DNS Name List`用于管理配置域名资产, `Combination` 用于组合具有相同属性及特征的静态资产并能以统一的方式进行展示.
<div align="center">
	<img src="/images/posts/tenable/sc-asset-add.png" height="" width="800">
</div>

* **Step 2** 点击 `Static IP List` 增加静态资产, 关于静态IP资产的增加建议从 `CMDB` 中进行关联同步.
<div align="center">
	<img src="/images/posts/tenable/sc-asset-add-2.png" height="" width="800">
</div>

* **Step 3** 点击 `Combination` 可以对静态资产列表进行重新组合定义，组合关键字 `AND`、`OR`，在`CMDB` 中对资产规划不明确的情况下使用组合资产可以对静态资产所属进行重新规划编排.
<div align="center">
	<img src="/images/posts/tenable/sc-asset-Combination.png" height="" width="800">
</div>

### <a name="sc-sec-baseline"></a>SC安全基线
　　`Tenable` 官方提供了大量的安全基线标准模板涵盖了网络设备、操作系统、中间件、数据库等软件与硬件, 基线模板标准如等保、CIS. 除了使用官方标准的安全基线模板, 也可以根据企业实际的检测标准来自定义基线检查标准, 在 `SC` 中所有的基线文件都使用xml格式定义检查项, 官方也提供了编写基线模板的例子详情可到 `Tenable` 官方进行查询. 基线检测的前提必须有资产的登陆认证, 所以该检查配置项与 `CMDB` 的关联更加重要.
<div align="center">
	<img src="/images/posts/tenable/sc-sec-baseline.png" height="" width="800">
</div>
基线模板
<div align="center">
	<img src="/images/posts/tenable/sc-sec-baseline-Template.png" height="" width="800">
</div>

* **Step 1** 点击 `Scans->Audit Files-Add`将已经准备好的安全基线模板上传并命名. 基线模板可以直接使用CIS.
<div align="center">
	<img src="/images/posts/tenable/sc-sec-baseline-auditfile.png" height="" width="800">
</div>

* **Step 2** 点击 `Scans->Policies-Add`添加扫描策略，`Tenable` 提供了影响力巨大的多个漏洞扫描检测策略其中包含了 `WannaCry`、`GHOST`等等, 这里使用的是 `Policy Compliance Auditing` 进行基线扫描策略配置, 基线扫描根据操作系统类型进行配置.
<div align="center">
	<img src="/images/posts/tenable/sc-policy-add.png" height="" width="800">
</div>

* **Step 3** 基线扫描策略建议按照一定规范命名方便建立扫描任务时进行关联策略, 策略命名后将准备好的基线模板进行关联.
<div align="center">
	<img src="/images/posts/tenable/sc-sec-baseline-add-2.png" height="" width="800">
</div>
关联模板
<div align="center">
	<img src="/images/posts/tenable/sc-sec-baseline-policy.png" height="" width="800">
</div>

* **Step 4** 配置并关联完成基线策略之后最重要的操作就是增加资产的登陆凭证，点击 `Scans->Credentails-Add`增加登陆认证, 登陆凭证必须与扫描的资产相对应.
<div align="center">
	<img src="/images/posts/tenable/sc-sec-baseline-credentials.png" height="" width="800">
</div>
###未完待续
