
# openEuler OpenStack 开发平台

openEuler OpenStack SIG成立于2021年，是由中国联通、中国电信、华为、统信等公司的开发者共同投入并维护的SIG小组，旨在openEuler之上提供原生的OpenStack，构建开放可靠的云计算技术栈，是openEuler的标杆SIG。但OpenStack本身技术复杂、包含服务众多，开发门槛较高，对贡献者的技术能力要求也较高，人力成本高居不下，在实际开发与贡献中存在各种各样的问题。为了解决SIG面临的问题，亟需一个openEuler+OpenStack解决方案，从而降低开发者门槛，降低投入成本，提高开发效率，保证SIG的持续活跃与可持续发展。

## 1. 概述

### 1.1 当前现状

目前，随着SIG的不断发展，我们明显的遇到了以下几类问题：
1. OpenStack技术复杂，涉及云IAAS层的计算、网络、存储、镜像、鉴权等方方面面的技术，开发者很难全知全会，提交的代码逻辑、质量堪忧。
2. OpenStack是由python编写的，python软件的依赖问题难以处理，以OpenStack Wallaby版本为例，涉及核心python软件包400+， 每个软件的依赖层级、依赖版本错综复杂，选型困难，难以形成闭环。
3. OpenStack软件包众多，RPM Spec编写开发量巨大，并且随着openEuler、OpenStack本身版本的不断演进，N:N的适配关系会导致工作量成倍增长，人力成本越来越大。
4. OpenStack测试门槛过高，不仅需要开发人员熟悉OpenStack，还要对虚拟化、虚拟网桥、块存储等Linux底层技术有一定了解与掌握，部署一套OpenStack环境耗时过长，功能测试难度巨大。并且测试场景多，比如X86、ARM64架构测试，裸机、虚机种类测试，OVS、OVN网桥测试，LVM、Ceph存储测试等等，更加加重了人力成本以及技术门槛。


### 1.2 解决方案

针对以上目前SIG遇到的问题，规范化、工具化、自动化的目标势在必行。本篇设计文档旨在在openEuler OpenStack SIG中提供一个端到端可用的开发解决方案，从技术规范到技术实现，提出严格的标准要求与设计方案，满足SIG开发者的日常开发需求，降低开发成本，减少人力投入成本，降低开发门槛，从而提高开发效率、提高SIG软件质量、发展SIG生态、吸引更多开发者加入SIG。主要动作如下：
1. 输出OpenStack服务类软件、依赖库软件的RPM SPEC开发规范，开发者及Reviewer需要严格遵守规范进行开发实施。
2. 提供OpenStack python软件依赖分析功能，一键生成依赖拓扑与结果，保证依赖闭环，避免软件依赖风险。
3. 提供OpenStack RPM spec生成功能，针对通用性软件，提供一键生成 RPM spec的功能，缩短开发时间，降低投入成本。
4. 提供自动化部署、测试平台功能，实现一键在任何openEuler版本上部署指定OpenStack版本的能力，快速测试、快速迭代。
5. 提供openEuler Gitee仓库自动化处理能力，满足批量修改软件的需求，比如创建代码分支、创建仓库、提交Pull Request等功能。

以上解决方法可以统一到一个系统平台中，我们称作OpenStack SIG Tool（以下简称oos），即就是openEuler OpenStack开发平台，具体架构如下：
```
            ┌────────────────────┐        ┌─────────────────────┐
            │         CLI        │        │         GUI         │
            └─────┬─────────┬────┘        └──────────┬──────────┘
                  │         │                        │
          Built-in│         └───────────┬────────────┘
                  │                     │REST
┌─────────────────▼─────────────────────▼────────────────────────────────┐
│                       OpenStack Develop Platform                       │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
          ┌────────────────────┬────┴─────────────┬────────────────┐
          │                    │                  │                │
┌─────────▼─────────┐  ┌───────▼───────┐  ┌───────▼───────┐  ┌─────▼─────┐
│Dependency Analysis│  │SPEC Generation│  │Deploy and Test│  │Code Action│
└───────────────────┘  └───────────────┘  └───────────────┘  └───────────┘
```
该架构主要有以下两种模式：
1. Client/Server模式
     在这种模式下，oos部署成Web Server形式，Client通过REST方式调用oos。
     - 优点：提供异步调用能力，支持并发处理，支持记录持久化。

     - 缺点：有一定安装部署成本，使用方式较为死板。

2. Built-in模式
     在这种模式下，oos无需部署，以内置CLI的方式对外提供服务，用户通过cli直接调用各种功能。
     - 优点：无需部署，随时随地可用。

     - 缺点：没有持久化能力，不支持并发，单人单用。

## 2. 详细设计

### 2.1 OpenStack Spec规范

Spec规范是一个或多个spec模板，针对RPM spec的每个关键字及构建章节，严格规定相关内容，开发者在编写spec时，必须满足规范要求，否则代码不允许被合入。规范内容由SIG maintainer公开讨论后形成结论，并定期审视更新。任何人都有权利提出对规范的质疑和建议， maintainer负责解释与刷新。规范目前包括两类：
1. 服务类软件规范
  此类软件以Nova、Neutron、Cinder等OpenStack核心服务为例，它们一般定制化要求高，内容区别大，必要人为手动编写。规范需清晰规定软件的分层方法、构建方法、软件包组成内容、测试方法、版本号规则等内容。

2. 通用依赖类软件规范
  此类软件一般定制化低，内容结构区别小，适合自动化工具一键生成，我们只需要在规范中定义相关工具的生成规则即可。

#### 2.1.1 服务类软件规范

OpenStack每个服务通常包含若干子服务，针对这些子服务，我们在打包的时候也要做拆包处理，分成若干个子RPM包。本章节规定了openEuler SIG对OpenStack服务的RPM包拆分的原则。

##### 2.1.1.1 通用原则

采用分层架构，RPM包结构如下图所示，以openstack-nova为例：

```
Level | Package                                                                       | Example
      |                                                                               |  
 ┌─┐  |                       ┌──────────────┐        ┌────────────────────────┐      | ┌────────────────────┐ ┌────────────────────────┐
 │1│  |                       │ Root Package │        │ Doc Package (Optional) │      | │ openstack-nova.rpm │ │ openstack-nova-doc.rpm │
 └─┘  |                       └────────┬─────┘        └────────────────────────┘      | └────────────────────┘ └────────────────────────┘
      |                                │                                              |
      |          ┌─────────────────────┼───────────────────────────┐                  |
      |          │                     │                           |                  |
 ┌─┐  | ┌────────▼─────────┐ ┌─────────▼────────┐                  |                  | ┌────────────────────────────┐ ┌────────────────────────┐
 │2│  | │ Service1 Package │ │ Service2 Package │                  |                  | │ openstack-nova-compute.rpm │ │ openstack-nova-api.rpm │
 └─┘  | └────────┬─────────┘ └────────┬─────────┘                  |                  | └────────────────────────────┘ └────────────────────────┘
      |          |                    |                            |                  |
      |          └──────────┬─────────┘                            |                  |
      |                     |                                      |                  |
 ┌─┐  |             ┌───────▼────────┐                             |                  | ┌───────────────────────────┐
 │3│  |             │ Common Package │                             |                  | │ openstack-nova-common.rpm │
 └─┘  |             └───────┬────────┘                             |                  | └───────────────────────────┘
      |                     │                                      |                  |
      |                     │                                      |                  |
      |                     │                                      |                  |
 ┌─┐  |            ┌────────▼────────┐            ┌────────────────▼────────────────┐ | ┌──────────────────┐ ┌────────────────────────┐
 │4│  |            │ Library Package ◄------------| Library Test Package (Optional) │ | │ python2-nova.rpm │ │ python2-nova-tests.rpm │
 └─┘  |            └─────────────────┘            └─────────────────────────────────┘ | └──────────────────┘ └────────────────────────┘
```

如图所示，分为4级

1. Root Package为总RPM包，原则上不包含任何文件。只做服务集合用。用户可以使用该RPM一键安装所有子RPM包。
    如果项目有doc相关的文件，也可以单独成包（可选）
2. Service Package为子服务RPM包，包含该服务的systemd服务启动文件、自己独有的配置文件等。
3. Common Package是共用依赖的RPM包，包含各个子服务依赖的通用配置文件、系统配置文件等。
4. Library Package为python源码包，包含了该项目的python代码。
    如果项目有test相关的文件，也可以单独成包（可选）

涉及本原则的项目有：

* openstack-nova
* openstack-cinder
* openstack-glance
* openstack-placment
* openstack-ironic

##### 2.1.1.2 特殊情况

有些openstack组件本身只包含一个服务，不存在子服务的概念,这种服务则只需要分为两级：

```
 Level | Package                                                         | Example
       |                                                                 |  
  ┌─┐  |               ┌──────────────┐  ┌────────────────────────┐      | ┌────────────────────────┐ ┌────────────────────────────┐
  │1│  |               │ Root Package │  │ Doc Package (Optional) │      | │ openstack-keystone.rpm │ │ openstack-keystone-doc.rpm │
  └─┘  |               └───────┬──────┘  └────────────────────────┘      | └────────────────────────┘ └────────────────────────────┘
       |                       |                                         |   
       |          ┌────────────┴───────────────────┐                     |
  ┌─┐  |  ┌───────▼─────────┐     ┌────────────────▼────────────────┐    | ┌──────────────────────┐ ┌────────────────────────────┐
  │2│  |  │ Library Package ◄-----| Library Test Package (Optional) │    | │ python2-keystone.rpm │ │ python2-keystone-tests.rpm │
  └─┘  |  └─────────────────┘     └─────────────────────────────────┘    | └──────────────────────┘ └────────────────────────────┘
```

1. Root Package RPM包包含了除python源码外的其他所有文件，包括服务启动文件、项目配置文件、系统配置文件等等。
    如果项目有doc相关的文件，也可以单独成包（可选）
2. Library Package为python源码包，包含了该项目的python代码。
    如果项目有test相关的文件，也可以单独成包（可选）

涉及本原则的项目有：

* openstack-keystone
* openstack-horizon

还有些项目虽然有若干子RPM包，但这些子RPM包是互斥的，则这种服务的结构如下：

```
Level | Package                                                                           | Example
      |                                                                                   |  
 ┌─┐  |                       ┌──────────────┐        ┌────────────────────────┐          | ┌───────────────────────┐ ┌───────────────────────────┐
 │1│  |                       │ Root Package │        │ Doc Package (Optional) │          | │ openstack-neutron.rpm │ │ openstack-neutron-doc.rpm │
 └─┘  |                       └────────┬─────┘        └────────────────────────┘          | └───────────────────────┘ └───────────────────────────┘
      |                                │                                                  |
      |          ┌─────────────────────┴───────────────────────────────┐                  |
      |          │                                                     |                  |
 ┌─┐  | ┌────────▼─────────┐ ┌──────────────────┐ ┌──────────────────┐ |                  | ┌──────────────────────────────┐ ┌───────────────────────────────────┐ ┌───────────────────────────────────┐
 │2│  | │ Service1 Package │ │ Service2 Package │ │ Service3 Package │ |                  | │ openstack-neutron-server.rpm │ │ openstack-neutron-openvswitch.rpm │ │ openstack-neutron-linuxbridge.rpm │
 └─┘  | └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘ |                  | └──────────────────────────────┘ └───────────────────────────────────┘ └───────────────────────────────────┘
      |          |                    |                    |           |                  |
      |          └────────────────────┼────────────────────┘           |                  |
      |                               |                                |                  |
 ┌─┐  |                       ┌───────▼────────┐                       |                  | ┌──────────────────────────────┐
 │3│  |                       │ Common Package │                       |                  | │ openstack-neutron-common.rpm │
 └─┘  |                       └───────┬────────┘                       |                  | └──────────────────────────────┘
      |                               │                                |                  |
      |                               │                                |                  |
      |                               │                                |                  |
 ┌─┐  |                      ┌────────▼────────┐      ┌────────────────▼────────────────┐ | ┌─────────────────────┐ ┌───────────────────────────┐
 │4│  |                      │ Library Package ◄------| Library Test Package (Optional) │ | │ python2-neutron.rpm │ │ python2-neutron-tests.rpm │
 └─┘  |                      └─────────────────┘      └─────────────────────────────────┘ | └─────────────────────┘ └───────────────────────────┘
```

如图所示，Service2和Service3互斥。

1. Root包只包含不互斥的子包，互斥的子包单独提供。
    如果项目有doc相关的文件，也可以单独成包（可选）
2. Service Package为子服务RPM包，包含该服务的systemd服务启动文件、自己独有的配置文件等。
    互斥的Service包不被Root包所包含，用户需要单独安装。
3. Common Package是共用依赖的RPM包，包含各个子服务依赖的通用配置文件、系统配置文件等。
4. Library Package为python源码包，包含了该项目的python代码。
    如果项目有test相关的文件，也可以单独成包（可选）

涉及本原则的项目有：

* openstack-neutron

#### 2.1.2 通用依赖类软件规范

一个依赖库一般只包含一个RPM包，不需要做拆分处理。

```
 Level | Package                                         | Example
       |                                                 |      
  ┌─┐  |  ┌─────────────────┐ ┌────────────────────────┐ | ┌──────────────────────────┐ ┌───────────────────────────────┐
  │1│  |  │ Library Package │ │ Help Package (Optional)│ | │ python2-oslo-service.rpm │ │ python2-oslo-service-help.rpm │
  └─┘  |  └─────────────────┘ └────────────────────────┘ | └──────────────────────────┘ └───────────────────────────────┘
```

**NOTE**

openEuler社区对python2和python3 RPM包的命名有要求，python2的包前缀为*python2-*，python3的包前缀为*python3-*。因此，OpenStack要求开发者在打Library的RPM包时，也要遵守openEuler社区规范。

### 2.2 软件依赖功能

软件依赖分析功能为用户提供一键分析目标OpenStack版本包含的全量python软件依赖拓扑及对应软件版本的能力。并自动与目标openEuler版本进行比对，输出对应的软件包开发建议。本功能包含两个子功能：
- 依赖分析

     对OpenStack python包的依赖树进行解析，拆解依赖拓扑。依赖树本质上是对有向图的遍历，理论上，一个正常的python依赖树是一个有向无环图，有向无环图的解析方法很多，这里采用常用的广度优先搜索方法即可。但在某些特殊场景下，python依赖树会变成有向有环图，例如：Sphinx是一个文档生产项目，但它自己的文档生成也依赖Sphinx，这就导致了依赖环的形成。针对这种问题，我们只需要把环上的特定节点手动断开即可。类似的还有一些测试依赖库。另一种规避方法是跳过文档、测试这种非核心库，这样不仅避免了依赖环的形成，也会极大减少软件包的数量，降低开发工作量。以OpenStack Wallaby版本为例，全量依赖包大概在700+以上，去掉文档、测试后，依赖包大概是300+左右。因此我们引入`core`核心的概念，用户根据自己的需求，选择要分析的软件范围。另外虽然OpenStack包含服务几十个，但用户可能只需要其中的某些服务，因此我们另外引入`projects`过滤器，用户可以根据自己的需求，指定分析的软件依赖范围。

- 依赖比对
    依赖分析完后，还要有对应的openEuler开发动作，因此我们还要提供基于目标openEuler版本的RPM软件包开发建议。openEuler与OpenStack版本之间有N:N的映射关系，一个openEuler版本可以支持多个OpenStack版本，一个OpenStack版本可以部署在多个openEuler版本上。用户在指定了目标openEuler版本和OpenStack版本后，本功能自动遍历openEuler软件库，分析并输出OpenStack涉及的全量软件包需要进行了操作，例如需要初始化仓库、创建openEuler分支、升级软件包等等。为开发者后续的开发提供指导。

#### 2.2.1 版本匹配规范

- 依赖分析

     输入：目标OpenStack版本、目标OpenStack服务列表、是否只分析核心软件

     输出：所有涉及的软件包及每个软件包的对应内容。格式如下：

     ```
     └──{OpenStack版本名}_cached_file
          └──packageA.yaml
          └──packageB.yaml
          └──packageC.yaml
          ......
     ```

     每个软件内容格式如下：

     ```
     {
        "name": "packageA", 
        "version_dict": {
           "version": "0.3.7",
           "eq_version": "",
           "ge_version": "0.3.5",
           "lt_version": "",
           "ne_version": [],
           "upper_version": "0.3.7"},
           "deep": {
              "count": 1,
              "list": ["packageB", "packageC"]},
           "requires": {}
     }
     ```

     关键字说明
     |       Key         | Description |
     |:-----------------:|:-----------:|
     |  name             | 软件包名                                         |
     | version_dict      | 软件版本要求，包括等于、大于等于、小于、不等于，等等 |
     | version_dict.deep | 表示该软件在全量依赖树的深度，以及深度遍历的路径     |
     | requires          | 包含本软件的依赖软件列表                           |


- 依赖比对
     输入：依赖分析结果、目标openEuler版本以及base比对基线

     输出：一个表格，包含每个软件的分析结果及处理建议，每一行表示一个软件，所有列名及定义规范如下：

     |       Column         | Description |
     |:-----------------:|:-----------:|
     |  Project Name     | 软件包名                                         |
     | openEuler Repo    | 软件在openEuler上的源码仓库名 |
     | Repo version | openEuler上的源码版本     |
     | Required (Min) Version         | 要求的最小版本                          |
     | lt Version | 要求小于的版本|
     | ne Version |要求的不等于版本|
     | Upper Version |要求的最大版本|
     | Status | 开发建议|
     | Requires | 软件的依赖列表|
     | Depth |软件的依赖树深度|
      
     其中`Status`包含的建议有:
     - “OK”：当前版本直接可用，不需要处理。
     - “Need Create Repo”：openEuler 系统中没有此软件包，需要在 Gitee 中的 src-openeuler repo 仓新建仓库。
     - “Need Create Branch”：仓库中没有所需分支，需要开发者创建并初始化。
     - “Need Init Branch”：表明分支存在，但是里面并没有任何版本的源码包，开发者需要对此分支进行初始化。
     - “Need Downgrade”：降级软件包。
     - “Need Upgrade”：升级软件包。

     开发者根据`Status`的建议进行后续开发动作。

#### 2.2.2 API和CLI定义

1. 创建依赖分析
     - CLI: `oos dependence analysis create`

     - endpoint: `/dependence/analysis`

     - type: POST

     - sync OR async: async

     - request body:
          ```
          {
               "release"[required]: Enum("OpenStack Relase"),
               "runtime"[optional][Default: "3.10"]: Enum("Python version"),
               "core"[optional][Default: False]: Boolean,
               "projects"[optional][Default: None]: List("OpenStack service")
          }
          ```
     - response body:
          ```
          {
               "ID": UUID,
               "status": Enum("Running", "Error")
          }
          ```
2. 获取依赖分析
     - CLI: `oos dependence analysis show`、`oos dependence analysis list`

     - endpoint: `/dependence/analysis/{UUID}`、`/dependence/analysis`

     - type: GET

     - sync OR async: sync

     - request body: None

     - response body:
          ```
          {
               "ID": UUID,
               "status": Enum("Running", "Error", "OK")
          }
          ```

3. 删除依赖分析
     - CLI: `oos dependence analysis delete`

     - endpoint: `/dependence/analysis/{UUID}`

     - type: DELETE

     - sync OR async: sync

     - request body: None

     - response body:
          ```
          {
               "ID": UUID,
               "status": Enum("Error", "OK")
          }
          ```

4. 创建依赖比对
     - CLI: `oos dependence generate`

     - endpoint: `/dependence/generate`

     - type: POST

     - sync OR async: async

     - request body:
          ```
          {
               "analysis_id"[required]: UUID,
               "compare"[optional][Default: None]: {
                    "token"[required]: GITEE_TOKEN_ID,
                    "compare-from"[optional][Default: master]: Enum("openEuler project branch"),
                    "compare-branch"[optional][Default: master]: Enum("openEuler project branch")

               }
          }
          ```
     - response body:
          ```
          {
               "ID": UUID,
               "status": Enum("Running", "Error")
          }
          ```

5. 获取依赖比对
     - CLI: `oos dependence generate show`、`oos dependence generate list`

     - endpoint: `/dependence/generate/{UUID}`、`/dependence/generate`

     - type: GET

     - sync OR async: sync

     - request body: None

     - response body:
          ```
          {
               "ID": UUID,
               "data" RAW(result data file)
          }
          ```

6. 删除依赖比对
     - CLI: `oos dependence generate delete`

     - endpoint: `/dependence/generate/{UUID}`

     - type: DELETE

     - sync OR async: sync

     - request body: None

     - response body:
          ```
          {
               "ID": UUID,
               "status": Enum("Error", "OK")
          }
          ```

### 2.3 软件SPEC生成功能
OpenStack依赖的大量python库是面向开发者的，这种库不对外提供用户服务，只提供代码级调用，其RPM内容构成单一、格式固定，适合使用工具化方式提高开发效率。

#### 2.3.1 SPEC生成规范

SPEC编写一般分为几个阶段，每个阶段有对应的规范要求：
1. 常规项填写，包括Name、Version、Release、Summary、License等内容，这些内容由目标软件的pypi信息提供
2. 子软件包信息填写，包括软件包名、编译依赖、安装依赖、描述信息等。这些内容也由目标软件的pypi信息提供。其中软件包名需要有明显的python化显示，比如以`python3-`为前缀。
3. 构建过程信息填写，包括%prep、%build %install %check内容，这些内容形式固定，生成对应rpm宏命令即可。
4. RPM包文件封装阶段，本阶段通过文件搜索方式，把bin、lib、doc等内容分别放到对应目录即可。

**NOTE**：在通用规范外，也有一些例外情况，需要特殊说明：
1. 软件包名如果本身已包含`python`这样的字眼，不再需要添加`python-`或`python3-`前缀。
2. 软件构建和安装阶段，根据软件本身的安装方式不同，宏命令包括`%py3_build`或`pyproject_build`，需要人工审视。
3. 如果软件本身包含C语言等编译类代码，则需要移除`BuildArch: noarch`关键字,并且在%file阶段注意RPM宏`%{python3_sitelib}`和`%{python3_sitearch}`的区别。

#### 2.3.2 API和CLI定义

1. 创建SPEC
     - CLI: `oos spec create`

     - endpoint: `/spec`

     - type: POST

     - sync OR async: async

     - request body:
          ```
          {
               "name"[required]: String,
               "version"[optional][Default: "latest"]: String,
               "arch"[optional][Default: False]: Boolean,
               "check"[optional][Default: True]: Boolean,
               "pyproject"[optional][Default: False]: Boolean,
          }
          ```
     - response body:
          ```
          {
               "ID": UUID,
               "status": Enum("Running", "Error")
          }
          ```

2. 获取SPEC
     -  CLI: `oos spec show`、`oos spec list`

     - endpoint: `/spec/{UUID}`、 `/spec/`

     - type: GET

     - sync OR async: sync

     - request body: None

     - response body:
          ```
          {
               "ID": UUID,
               "status": Enum("Running", "Error", "OK")
          }
          ```

3. 更新SPEC
     - CLI: `oos spec update`

     - endpoint: `/spec/{UUID}`

     - type: POST

     - sync OR async: async

     - request body:
          ```
          {
               "name"[required]: String,
               "version"[optional][Default: "latest"]: String,
          }
          ```

     - response body:
          ```
          {
               "ID": UUID,
               "status": Enum("Running", "Error")
          }
          ```

4. 删除SPEC
     - CLI: `oos spec delete`

     - endpoint: `/spec/{UUID}`

     - type: DELETE

     - sync OR async: sync

     - request body: None

     - response body:
          ```
          {
               "ID": UUID,
               "status": Enum("Error", "OK")
          }
          ```

### 2.4 自动化部署、测试功能

OpenStack的部署场景多样、部署流程复杂、部署技术门槛较高，为了解决门槛高、效率低、人力多的问题，openEuler OpenStack开发平台需要提供自动化部署、测试功能。

- 自动化部署

     提供基于openEuler的OpenStack的一键部署能力，包括支持不同架构、不同服务、不同场景的部署功能，提供基于不同环境快速发放、配置openEuler环境的能力。并提供`插件化`能力，方便用户扩展支持的部署后端和场景。

- 自动化测试

     提供基于openEuler的OpenStack的一键测试能力，包括支持不同场景的测试，提供用户自定义测试的能力，并规范测试报告，以及支持对测试结果上报和持久化的能力。

#### 2.4.1 自动化部署

自动化部署主要包括两部分：openEuler环境准备和OpenStack部署。

- openEuler环境准备

  提供快速发放openEuler环境的能力，支持的发放方式包括`创建公有云资源`和`纳管已有环境`，具体设计如下：

  ```
  **NOTE**
     openEuler的OpenStack支持以RPM + systemd的方式为主，暂不支持容器方式。
  ```

  - 创建公有云资源

    创建公有云资源以虚拟机支持为主（裸机在云上操作负责，生态满足度不足，暂不做支持）。采用插件化方式，提供多云支持的能力，以华为云为参考实现，优先实现。其他云的支持根据用户需求，持续推进。根据场景，支持all in one和三节点拓扑。
     1. 创建环境
          - CLI: `oos env create`

          - endpoint: `/environment`

          - type: POST

          - sync OR async: async

          - request body:
               ```
               {
                    "name"[required]: String,
                    "type"[required]: Enmu("all-in-one", "cluster"),
                    "release"[required]: Enmu("openEuler_Release"),
                    "flavor"[required]： Enmu("small", "medium", "large"),
                    "arch"[required]： Enmu("x86", "arm64"),
               }
               ```

          - response body:
               ```
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

     2. 查询环境
          - CLI: `oos env list`

          - endpoint: `/environment`

          - type: GET

          - sync OR async: async

          - request body: None

          - response body:
               ```
               {
                    "ID": UUID,
                    "Provider": String,
                    "Name": String,
                    "IP": IP_ADDRESS,
                    "Flavor": Enmu("small", "medium", "large"),
                    "openEuler_release": String,
                    "OpenStack_release": String,
                    "create_time": TIME,
               }
               ```
     3. 删除环境
          - CLI: `oos env delete`

          - endpoint: `/environment/{UUID}`

          - type: DELETE

          - sync OR async: sync

          - request body: None

          - response body:
               ```
               {
                    "ID": UUID,
                    "status": Enum("Error", "OK")
               }
               ```

  - 纳管已有环境

    用户还可以直接使用已有的openEuler环境进行OpenStack部署，需要把已有环境纳管到平台中。纳管后，环境与创建的项目，可以直接查询或删除。
    1. 纳管环境
          - CLI: `oos env manage`

          - endpoint: `/environment/manage`

          - type: POST

          - sync OR async: sync

          - request body:
               ```
               {
                    "name"[required]: String,
                    "ip"[required]: IP_ADDRESS,
                    "release"[required]: Enmu("openEuler_Release"),
                    "password"[required]： String,
               }
               ```

          - response body:
               ```
               {
                    "ID": UUID,
                    "status": Enum("Error", "OK")
               }
               ```

- OpenStack部署

     提供在已创建/纳管的openEuler环境上部署指定OpenStack版本的能力。
     1. 部署OpenStack
          - CLI: `oos env setup`

          - endpoint: `/environment/setup`

          - type: POST

          - sync OR async: async

          - request body:
               ```
               {
                    "target"[required]: UUID(environment),
                    "release"[required]: Enmu("OpenStack_Release"),
               }
               ```

          - response body:
               ```
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

     2. 初始化OpenStack资源
          - CLI: `oos env init`

          - endpoint: `/environment/init`

          - type: POST

          - sync OR async: async

          - request body:
               ```
               {
                    "target"[required]: UUID(environment),
               }
               ```

          - response body:
               ```
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

     3. 卸载已部署OpenStack
          - CLI: `oos env clean`

          - endpoint: `/environment/clean`

          - type: POST

          - sync OR async: async

          - request body:
               ```
               {
                    "target"[required]: UUID(environment),
               }
               ```

          - response body:
               ```
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

#### 自动化测试

环境部署成功后，SIG开发平台提供基于已部署OpenStack环境的自动化测试功能。主要包含以下几个重要内容：

OpenStack本身提供一套完善的测试框架。包括`单元测试`和`功能测试`，其中`单元测试`在`2.3章节`中已经由RPM spec包含，spec的%check阶段可以定义每个项目的单元测试方式，一般情况下只需要添加`pytest`或`stestr`即可。`功能测试`由OpenStack Tempest服务提供，在上文所述的自动化部署`oos env init`阶段，oos会自动安装Tempest并生成默认的配置文件。
- CLI: `oos env test`

- endpoint: `/environment/test`

- type: POST

- sync OR async: async

- request body:
     ```
     {
          "target"[required]: UUID(environment),
     }
     ```

- response body:
     ```
     {
          "ID": UUID,
          "status": Enum("Running", "Error")
     }
     ```

测试执行完后，oos会输出测试报告，默认情况下，oos使用`subunit2html `工具，生成html格式的Tempest测试结果文件。

### 2.5 openEuler自动化开发功能

OpenStack涉及软件包众多，随着版本不断地演进、支持服务不断的完善，SIG维护的软件包列表会不断刷新，为了降低重复的开发动作，oos还封装了一些易用的代码开发平台自动化能力，比如基于Gitee的自动代码提交能力。功能如下：
```
     ┌───────────────────────────────────────────────────┐
     │                     Code Action                   │
     └─────────────────────┬─────────────────────────────┘
                           │
           ┌───────────────┼───────────────────┐
           │               │                   │
     ┌─────▼─────┐  ┌──────▼──────┐  ┌─────────▼─────────┐
     │Repo Action│  │Branch Action│  │Pull Request Action│
     └───────────┘  └─────────────┘  └───────────────────┘
```


1. `Repo Action`提供与软件仓相关的自动化功能：

     1. 自动建仓
          - CLI: `oos repo create`

          - endpoint: `/repo`

          - type: POST

          - sync OR async: async

          - request body:
               ```
               {
                    "project"[required]: String,
                    "repo"[required]: String,
                    "push"[optional][Default: "False"]: Boolean,
               }
               ```

          - response body:
               ```
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```


2. `Branch Action`提供与软件分支相关的自动化功能：

     1. 自动创建分支
          - CLI: `oos repo branch-create`

          - endpoint: `/repo/branch`

          - type: POST

          - sync OR async: async

          - request body:
               ```
               {
                    "branches"[required]: {
                         "branch-name"[required]: String,
                         "branch-type"[optional][Default: "None"]: Enum("protected"),
                         "parent-branch"[required]: String
                    }
               }
               ```

          - response body:
               ```
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

3. `Pull Request Action`提供与代码PR相关的自动化功能：

     1. 新增PR评论，方便用户执行类似`retest`、`/lgtm`等常规化评论。
          - CLI: `oos repo pr-comment`

          - endpoint: `/repo/pr/comment`

          - type: POST

          - sync OR async: sync

          - request body:
               ```
               {
                    "repo"[required]: String,
                    "pr_number"[required]: Int,
                    "comment"[required]: String
               }
               ```

          - response body:
               ```
               {
                    "ID": UUID,
                    "status": Enum("OK", "Error")
               }
               ```

     2. 获取SIG所有PR，方便maintainer获取当前SIG的开发现状，提高评审效率。
          - CLI: `oos repo pr-fetch`

          - endpoint: `/repo/pr/fetch`

          - type: POST

          - sync OR async: async

          - request body:
               ```
               {
                    "repo"[optional][Default: "None"]: List[String]
               }
               ```

          - response body:
               ```
               {
                    "ID": UUID,
                    "status": Enum("Running", "Error")
               }
               ```

## 3. 质量、安全与合规
SIG开源软件需要符合openeEuler社区对其中软件的各种要求，并且也要符合OpenStack社区软件的出口标准。

### 3.1 质量与安全

- 软件质量（可服务性）
     1. 对应软件代码需包含单元测试，覆盖率不低于80%。
     2. 需提供端到端功能测试，覆盖上述所有接口，以及核心的场景测试。
     3. 基于openEuler社区CI，构建CI/CD流程，所有Pull Request要有CI保证代码质量，定期发布release版本，软件发布间隔不大于3个月。
     4. 基于Gitee ISSUE系统处理用户发现并反馈的问题，闭环率大于80%，闭环周期不超过1周。

- 软件安全
     1. 数据安全：软件全程不联网，持久存储中不包含用户敏感信息。
     2. 网络安全：OOS在REST架构下使用http协议通信，但软件设计目标实在内网环境中使用，不建议暴露在公网IP中，如必须如此，建议增加访问IP白名单限制。
     3. 系统安全：基于openEuler安全机制，定期发布CVE修复或安全补丁。
     4. 应用层安全：不涉及，不提供应用级安全服务，例如密码策略、访问控制等。
     5. 管理安全：软件提供日志生成和周期性备份机制，方便用户定期审计。

- 可靠性

     本软件面向openEuler社区OpenStack开发行为，不涉及服务上线或者商业生产落地，所有代码公开透明，不涉及私有功能及代码。因此不提供例如节点冗余、容灾备份能功能。

### 3.2 合规
1. License合规

     本平台采用Apache2.0 License，不限制下游fork软件的闭源与商业行为，但下游软件需标注代码来源以及保留原有License。

2. 法务合规

     本平台由开源开发者共同开发维护，不涉及商业公司的秘密以及非公开代码。所有贡献者需遵守openEuler社区贡献准则，确保自身的贡献合规合法。SIG及社区本身不承担相应责任。

     如发现不合规的源码，SIG无需获取贡献者的允许，有权利及义务及时删除。并有权禁止不合规代码或开发者继续贡献。

     开发者如果有非公开代码需要贡献，则要先遵守本公司的开源流程与规定，并按照openEuler社区开源规范公开贡献代码。

## 4. 实施计划

|       时间         | 内容 | 状态|
|:-----------------:|:-----------:|:-----------:|
|  2021.06           | 完成软件整体框架编写，实现CLI Built-in机制，至少一个API可用 | Done |
|  2021.12           | 完成CLI Built-in机制的全量功能可用 | Done |
|  2022.06           | 完成质量加固，保证功能，在openEuler OpenStack社区开发流程中正式引入OOS | Done |
|  2022.12           | 不断完成OOS，保证易用性、健壮性，自动化覆盖度超过80%，降低开发人力投入 | Done |
|  2023.06           | 补齐REST框架、CI/CD流程，丰富Plugin机制，引入更多backend支持 | Working in progress |
|  2023.12           | 完成前端GUI功能 | Planning |
