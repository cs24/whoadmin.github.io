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

* [SC安装](#Where-to-go-from-here)
* [Nessus漏洞扫描器引擎安装](#Where-to-go-from-here)
* [SC关联并授权Nessus漏洞扫描器](#Where-to-go-from-here)
* [SC组织创建](#Where-to-go-from-here)
* [SC数据仓库概念及区域性划分](#Where-to-go-from-here)
* [SC扫描空间概念及区域性划分](#Where-to-go-from-here)
* [SC资产管理](#Where-to-go-from-here)
* [SC对Nessus漏洞扫描器的管理](#Where-to-go-from-here)
* [SC安全基线](#Where-to-go-from-here)
* [SC漏洞扫描](#Where-to-go-from-here)
* [SC仪表板](#Where-to-go-from-here)
* [SC简单漏洞运营](#Where-to-go-from-here)


### <a name="When-to-apply-neural-net"></a>SecurityCenter安装

　　`SecurityCenter` 的安装包需要在[Tenable官方网站](https://www.tenable.com/downloads)进行下载, 由于方便运维团队进行自动化管理, 本文Tenable的运行环境为 `Centos 7.x` 或 `Centos 6.x`, 所以下载的SC安装包为rpm包。Tenable 官方将 `SC` 定位成一个漏洞数据集成管理中心, 它本身不具备主动扫描漏洞的能力, 所以下载SC安装包的同时还需要下载Nessus的rpm安装包, SC与Nessus是收费版软件要想使用必须先购买license, 购买了SC的license之后就可以直接对Nessus进行授权管理, 一个SC的license可以授权管理多少的Nessus这里就不多谈, 小伙伴们可以自己去了解。

　　关于的使用 `SC` 我这里只能讲解已经在使用或者已经测试过的一些功能, 还有部分功能如主机安全等并未购买相关服务及功能这里无法细说。 通过几个月对 `SC` 的使用发现这个产品的特点及局限性有了一个大致的了解, 现在让我们来安装它：

* **首先，需要在[Tenable官方](https://www.tenable.com/downloads)下载SC与Nessus的安装包**，由于我这里的网络环境复杂且需要安装管理的Nessus扫描器 (这里称为Nessus节点) 较多, 考虑到开启大量扫描时SC的性能分配给SC的配置为8core、32G、1TB (IDC规模更大的话可以增加配置) 作为管理节点运行, Nessus的数量可以按照业务与服务器体量的大小来进行安装配置。 由于SC只是通过Nessus进行漏洞扫描,使用SC调用Nessus扫描时Nessus本地不会保存扫描结果数据, 所以分配给Nessus的资源可以参考对单个节点Nessus使用情况来分配。 


# 未完待续