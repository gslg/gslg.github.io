title: Linux RPM包及yum源制作
author: gslg
tags:
  - rpm
  - yum
categories:
  - linux
date: 2019-06-26 09:35:00
---
## 概述
RPM是Red Hat Enterprise Linux, CentOS, and Fedora等linux的包管理器，使用RPM可以很容易的发布，管理，更新这些系统上的软件.

使用RPM可以做如下事情:   
- **安装，重装，卸载，验证软件包**  
 用户可以使用标准的包管理工具(Yum 或者 PackageKit)来安装，重装，卸载或者验证软件包
- **使用已安装包的数据库来查询或者验证**  
RPM维护了一个数据库来存储已安装的包和它们的文件,用户可以很容易查询和验证自己系统上安装的软件包
- **使用元数据来描述软件包，以及安装手册等**  
每个RPM包都包含一个元数据用来描述该包的组件，版本，发布，大小，工程的URL，安装介绍等信息
- **将原始的软件源打包成源代码包和二进制包**  
RPM允许将原始的软件源码打包成源码包或者二进制包提供给你的用户.在源代码包中，你有原始的源代码以及使用的任何补丁，以及完整的构建说明.随着软件新版本发布，这种设计简化了软件包的维护。
- **添加包到yum仓库**  
可以添加你的RPM包到yum仓库，这样用户就可以很容易的找到并部署你的软件
- **对你的软件包进行数字签名**  
使用GPG签名密钥，你可以对包进行数字签名，以便用户能够验证包的真实性。

## RPM包
这里主要描述RPM制作的环境准备，以及一些基础概念，[**高级主题**](https://rpm-packaging-guide.github.io/#advanced-topics)在这里可以找到.
### 环境准备
为了制作我们的RPM包，我们需要在自己的linux机器上安装相关工具和环境,这里以Centos7为例.  
```shell 
$ yum install gcc rpm-build rpm-devel rpmlint make python bash coreutils diffutils patch rpmdevtools
```
如果是Fedora
```shell
$ dnf install gcc rpm-build rpm-devel rpmlint make python bash coreutils diffutils patch rpmdevtools
```
### 什么是RPM包
RPM package就是一个简单的文件，该文件包含了系统所需要的一些其他文件以及对这些文件的信息。具体来说，一个RPM包由包含文件的[CPIO归档文件](https://en.wikipedia.org/wiki/Cpio)和包含该包元数据的RPM头组成。rpm包管理就是通过该元数据来决定包的依赖,安装路径以及其他信息.  

有两类的RPM包:  
- RPM源码包(source RPM),即`SRPM`
- 二进制RPM包(binary RPM)  

源码rpm包和二进制RPM包共享文件格式和工具，但是具有不同的内容和作用.`SRPM`包含了源码，可选的补丁以及`SPEC`文件，`SPEC`文件描述了如何将该`SRPM`构建为`binary RPM`.  
`binary RPM`包含了从源码以及补丁构建的二进制文件

### RPM打包工具
在上面环境准备过程中,已经安装了RPM打包过程中可能用到的工具，可以通过下面的命令来获取工具列表:  
```shell
$ rpm -ql rpmdevtools | grep bin
```
### RPM工作空间
类似于我们开发程序，有个工作空间来存放我们的源码，编译文件等，制作RPM也需要工作空间。首次使用时，我们可以使用`rpmdev-setuptree`来初始化工作空间:  
```shell
$ rpmdev-setuptree

$ tree ~/rpmbuild/
/home/rpmuser/rpmbuild/
|-- BUILD
|-- RPMS
|-- SOURCES
|-- SPECS
`-- SRPMS
```
可以看到`rpmdev-setuptree`帮我们在当前用户的目录下生成了rpm的工作空间,这些目录的作用如下:  

|目录|作用|补充|
|:--:|:--||
|BUILD|当构建包时，在此创建各种%buildRoot目录。如果日志输出没有提供足够的信息,该目录对排错也很有用.|我们对源码压缩包进行构建rpm时,解压后的文件就是放在这个目录的|
|RPMS|我们制作的Binary RPM就存放在这里,在这个目录中会根据系统架构创建不同的子目录,比如`x86_64`和`noarch`,我们的RPM包就放在对应的子目录下|生成`x86_64`或者`noarch`类型的rpm可以通过`SPEC`文件指定|
|SOURCES|这里是打包用户放置源码压缩包或补丁的目录，`rpmbuild`命令就是在这里找源码包的|一般我们会将源码包压缩为`.tar.gz`格式，然后放在这个目录进行RPM构建|
|SPECS|放置SPEC文件|每制作一个应用的rpm包,就需要对应一个SPEC文件，该文件就是放在这个目录|
|SRPMS|当使用`rpmbuild`构建RPM源码包时，构建的SRPM就是放在这的|构建源码包的目的就是和当前系统架构解耦|

### 什么是SPEC文件
`SPEC`文件可以看做是一个"菜谱",`rpmbuild`工具使用该"菜谱"来实际构建RPM.它通过在一系列`section`中定义指令来告诉构建系统要做什么.`sections`在前言(`Preamble`)和正文(`Body`)中定义。前言包含正文中使用的一系列元数据项，正文包含指令的主要部分。
#### 前言指令条目
下表列出了前言中可能包含的一些指令条目:  

|SPEC 指令 | 描述 |  
|:--|:--|  
| ``Name`` | 包的基本名字,应该和SPEC文件名保持一致|  
| ``Version`` | 软件的上游版本号|
| ``Release`` | 此版本软件发布的次数,初始值设置为 1%{?dist},并且在新发布时增加它. 当该软件新的 ``Version``构建时将该值重置为1.|
| ``Summary`` | 一行简明扼要的概述|
| ``License`` | 正在打包的软件的许可证。对于在社区发行版（如Fedora）中分发的软件包，这必须是一个符合特定发行版许可指南的开源许可证。|
| ``URL`` |有关程序的详细信息的完整URL。大多数情况下，这是正在打包的软件的上游项目网站。|
| ``Source0`` | 上游源代码压缩存档的路径或url(不含补丁)。这应该指向存档的可访问和可靠存储，例如，上游页面，而不是打包程序的本地存储。如果需要，可以添加更多的sourcex指令，每次增加数字，例如：source1、source2、source3等等。|
| ``Patch0`` | 源码的补丁，如果需要，可以添加更多，例如:Patch1, Patch2,Patch3等等|
| ``BuildArch`` | 如果RPM包不依赖于架构,例如,如果整个使用解释型语言开发,可以设置为``BuildArch: noarch``.如果不设置，那么就会检测该系统的架构，例如``x86_64``.
| ``BuildRequires`` | 逗号或者空格分隔的软件包，主要用来编译由编译型语言编写的程序. 可以有多个``BuildRequires``节点,每一行列出一个.|
| ``Requires`` | 该软件包安装后运行时依赖的软件包，以逗号和空格分隔. 和``BuildRequires``一样，也可以有多个节点，每一行一个.|
| ``ExcludeArch`` | 如果该软件不能在特定的处理器体系结构上运行，可以在这里排除该体系结构.|

`SPEC`文件中的`Name`,`Version`,`Release`指令构成了该RPM包的文件名，RPM包的维护人员或者系统管理者通常把这三个指令叫做**N-V-R**或者**NVR**,因为RPM包的名字就是**NAME-VERSION-RELEASE**这样的格式.  

例如我们查看Python的RPM包,就可以发现这种命名格式
```shell
$ rpm -q python
python-2.7.5-77.el7_6.x86_64
```
这里，`python`对应于rpm包的`Name`,`2.7.5`对应于`Version`,`34.el7_6`对应于`Release`,最后一个标记是x86_64，它表示体系结构.与NVR不同的是，体系结构不是由打包者直接控制的,而是由`rpmbuild`构建环境定义的.除了与架构无关的`noarch` rpm包是个例外.

#### 正文指令条目
下表是`SPEC`文件正文部分的指令条目  

|SPEC 指令 | 描述 |  
|:--|:--|  
| ``%description`` |该RPM打包软件的完整描述。此描述可以跨多行，可以分为多个段落。|  
| ``%prep`` | 用于准备要构建的软件的命令或一系列命令，例如，在source0中解包归档文件。此指令可以包含shell脚本。|
| ``%build`` | 一条或多条命令,用来将软件编译为机器码(某些编译型语言)或者字节码(某些解释型语言) |
| ``%install`` | 一条或多条命令，用于将所需的生成项目从%builddir（构建发生的位置）复制到%buildroot目录（包含要打包文件的目录结构）。实际上就是拷贝`~/rpmbuild/BUILD`中的文件到`~/rpmbuild/BUILDROOT`中，并在`~/rpmbuild/BUILDROOT`中创建必要的目录.这些动作只会发生在构建包的时候，在终端用户使用rpm包时并不会发生.详细参考 [working-with-spec-files](https://rpm-packaging-guide.github.io/#working-with-spec-files)|
| ``%check`` | 一条或多条用于测试该软件的命令,例如包含几个单元测试|
| ``%files`` | 需要安装到客户系统的文件列表|
| ``%changelog`` | 在不同`Version`和`Release`之间发生变化的记录|
 
 
#### 高级指令条目
除了上述一些基本的指令外，`SPEC`文件也可以包含一些高级的指令,例如一些脚本或者触发器，它们可以在终端用户安装过程中的不同阶段进行执行,(不是在rpm构建过程中执行).  
查看[triggers-and-scriptlets](https://rpm-packaging-guide.github.io/#triggers-and-scriptlets)获取高级主题

### BuildRoots
在rpm打包的上下文中,"buildroot"是一个[Chroot](https://en.wikipedia.org/wiki/Chroot)环境.这意味着构建工件将使用与最终用户系统中相同的文件系统层次结构放置在这里，其中“buildroot”充当根目录,构建工件的放置应该符合最终用户系统的文件系统层次结构标准。  

"buildroot"文件后面会被放进[cpio归档文件](https://en.wikipedia.org/wiki/Cpio),作为RPM的一个重要组成部分.当在最终用户的系统上安装该RPM包时，这些文件将被提取到根目录中，从而保持正确的层次结构。

> !Note: 可以在SPEC文件中使用%{buildroot}来引用该根目录  
> 关于Chroot也可以看看[Linux Chroot](https://www.imooc.com/article/26318)  

### RPM Macros
[RPM Macros](http://rpm.org/user_doc/macros.html)就是一个纯文本替换,这样可以在重复使用某个值时直接引用它，然后让rpm替你替换它.

例如，在我们写SPEC文件的过程中，我们可能在多个地方引用`Version`这个值，我们可以定义`Version`在`%{version}`宏中,这样就可以在后续使用`%{version}`来引用该值

我们可以通过`rpm --eval %{_MACRO}`来计算已定义的宏,例如:

```shell
$ rpm --eval %{_bindir}
/usr/bin

$ rpm --eval %{_libexecdir}
/usr/libexec
```
```shell
# On a RHEL 7.x machine
$ rpm --eval %{?dist}
.el7

# On a Fedora 23 machine
$ rpm --eval %{?dist}
.fc23
```

已经预定义了很多宏,详情查看[more-on-macros](https://rpm-packaging-guide.github.io/#more-on-macros)

## 示例
到这里，我们基本熟悉了RPM的基础概念以及准备工作，下面我们就开始使用SPEC文件来构建我们的RPM包
#### 源码包准备
##### bash脚本
- 在我们的工作空间创建`bello.sh`
```shell
#!/bin/bash
printf "Hello World\n"
```
- 我们把`bello.sh`变为可执行的
```shell
$ chmod +x bello.sh
$ ./bello
Hello World
```
- 我们再随便建一个LICENSE文件:
 ```shell  
 $echo GPLv3+ > LICENSE
 $tree
   |─ bello
     |─ bello.sh
    |─ LICENSE

 ```
- 打包我们的源码,一般压缩包的命名方式为**name-version.tar.gz**
```shell
$ tar -cvzf bello-0.1.tar.gz bello
bello/
bello/LICENSE
bello/bello.sh
```
- 将源码包拷贝至`rpmbuild`工作空间的`SOURCES`子目录下
```shell
$ mv bello-0.1.tar.gz ~/rpmbuild/SOURCES/
```
##### C程序包
 1. cello
  ```
   $ mkdir cello && cd cello
   $ vim cello.c
  #include <stdio.h>
  int main(){
    printf("Hello world.\n");
    return 0;
  }
 ```
2.因为C程序需要编译安装，所以再写个`Makefile`:
 {% codeblock %}
     cello:
            gcc -g -o cello cello.c

    clean:
            rm cello

    install:
            mkdir -p $(DESTDIR)/usr/bin
            install -m 0755 cello $(DESTDIR)/usr/bin/cello
 {% endcodeblock %}
##### Java Spring Boot程序包
##### 前端包
到这里，我们已经准备好了我们的源码包,下面就开始写`SPEC`文件  

#### Working with SPEC files 
源码包准备好以后，我们开始编写`SPEC`文件，在前面我们提到,`SPEC`文件实际是构建RPM包的"菜谱",为了构建RPM包，我们大部分的工作就是编写`SPEC`文件  

每构建一个新的软件包，我们首先需要创建一个对应额SPEC文件，手动写SPEC文件比较麻烦，我们可以使用`rpmdev-newspec`工具创建一个SPEC文件模板，我们只需要填充必要的部分.

##### 创建bello SPEC文件
```shell
 $ cd ~/rpmbuild/SPECS
 $ rpmdev-newspec bello
  bello.spec created; type minimal, rpm version >= 4.11. 
```
我们可以看一下生成的spec文件内容:  
 {% codeblock bello.spec %}
   Name:           bello  
   Version:        
   Release:        1%{?dist}  
   Summary:        

   License:        
   URL:            
   Source0:        

   BuildRequires:  
   Requires:       

   %description  


   %prep  
   %setup -q


   %build  
   %configure  
   make %{?_smp_mflags}


   %install  
   rm -rf $RPM_BUILD_ROOT  
   %make_install


   %files  
   %doc  

   %changelog   
 {% endcodeblock %}
 
 ##### 修改bello.SPEC文件
 1. 填充`Name`, `Version`, `Release`, 和 `Summary`条目
   - `Name`已经通过`rpmdev-newspec`参数指定了
   - 设置`Version`,与bello源码的上游版本号相匹配，这里是`0.1`
   - 这里`Release`已经被自动设置为初始值`1`.当需要更新这个包但是上游版本号不变时，例如补丁更新,需要递增该值.当上游一个新版本发布时，例如发布`bello-0.2`，那么需要将该值重置为`1`. `%{?dist}`我们在上面的宏部分已经说过了.  
   - `Summary`写一行简明的概述  
在修改后，bello.spec文件长这样:  
{% codeblock bello.spec %}
Name:           bello
Version:        0.1
Release:        1%{?dist}
Summary:        Hello World example implemented in bash script
{% endcodeblock %}    

 2. 填充`License`, `URL`, 和`Source0` 条目:
  - `License`文件格式可以参考[Fedora License Guidelines](https://fedoraproject.org/wiki/Licensing:Main),这里我们使用`GPLv3+`
  - `URL`提供了到软件主页的访问地址.例如`https://example.com/bello`,一般使用` %{name}`宏来代替，如`https://example.com/%{name}`
  - `Source0`提供了到源码的访问路径.它应该直接指向正在需要打包的软件，例如`https://example.com/bello/releases/bello-0.1.tar.gz`,使用宏替代则为:`https://example.com/%{name}/releases/%{name}-%{version}.tar.gz`.  
  在我们修改后，长这样:
  {% codeblock bello.spec %}
  Name:           bello
  Version:        0.1
  Release:        1%{?dist}
  Summary:        Hello World example implemented in bash script

  License:        GPLv3+
  URL:            https://example.com/%{name}
  Source0:        https://example.com/%{name}/release/%{name}-% {version}.tar.gz
  {% endcodeblock %}
  
 3. 填充`BuildRequires`和`Requires`以及`BuildArch`条目
  - `BuildRequires`指定了软件包构建时期的依赖,对于bello来说，没有构建这一步骤，因为bello是通过解释型语言编写的，只需要将该文件安装到客户本地文件系统上即可.因此这里直接删除该条目即可.
  - `Requires`指定了软件运行时期的依赖项，因为我们的bello只依赖于`bash`执行环境,因此只需要添加`bash`即可.
  - 因为bello程序是通过解释型语言编写的，不需要编译，因此需要设置`BuildArch`为`noarch`,这样就告诉RPM构建包时不需要去绑定打包机器的处理器架构.  
  编辑完成后，我们的spec文件长这样了:
  {% codeblock bello.spec %}
  Name:           bello
  Version:        0.1
  Release:        1%{?dist}
  Summary:        Hello World example implemented in bash script

  License:        GPLv3+
  URL:            https://example.com/%{name}
  Source0:        https://example.com/%{name}/release/%{name}-%{version}.tar.gz

  Requires:       bash

  BuildArch:      noarch
  {% endcodeblock %}
  
 4. 填充`%description`,`%prep`, `%build`, `%install`, `%files`,和`%changelog`条目。这些条目相当于一个章节标题,因为可以有多行，多个指令或者多个脚本任务执行。
  - `%description`是一段长的，完整的关于该软件的描述,可以包含一个或多个段落
  - `%prep`部分指定了如何去准备构建环境.通常涉及扩展源代码的压缩文档，补丁的应用，以及可能解析源代码中提供的信息，以便在SPEC的后续部分中使用。例子中我们简单使用内置的宏`%setup -q`
  - `%build`部分指定了怎样去构建我们的软件. 因为`bash`不需要构建，因此这里直接删除模板中提供的，留一个空行就行了
  - `%install`部分规定了`rpmbuild`在构建软件后将其安装到`BUILDROOT`目录下的操作.该目录是一个空的[chroot](https://en.wikipedia.org/wiki/Chroot)根目录，类似于最终用户的根目录。 在这里，我们应该创建任何包含已安装文件的目录。  
  因为安装`bello`仅仅需要创建一个目标文件夹，然后把可执行的bash脚本安装在该目录即可，我们会使用`install`命令来完成该操作.RPM的宏允许我们能够完成这些操作而不需要硬编码我们的安装路径.  
  这样，我们修改后的`%install`部分:
  {% codeblock bello.spec %}
     %install

    mkdir -p %{buildroot}/%{_bindir}

    install -m 0755 %{name} %{buildroot}/%{_bindir}/%{name}
  {% endcodeblock %}
  - `%files`部分列出了RPM提供的文件列表,以及安装到终端用户系统中的全路径.因此，`bello`安装包只提供了一个`/usr/bin/bello.sh`文件需要安装,使用宏代替则为`%{_bindir}/{name}`  
  在这个部分中，我们可以使用内置的宏指定标识文件的角色,这样的好处在于可以通过rpm命令来检索这些元数据.例如，为了表明**LICENSE**文件是一个软件的`license file`,我们可以使用宏`%license`来标明:
  ```shell
  %files
%license LICENSE
%{_bindir}/%{name}
  ```
 5. `%changelog`，这是最后一个部分.是包的每个版本发布的加盖时间戳的条目列表，它们记录了包的变化,而不是软件的变化，例如添加补丁，或者更改`%build`等构建程序.
  第一行的格式:
  ```
  * Day-of-Week Month Day Year Name Surname <email> - Version-Release
  ```
 实际变化的节点格式:
  - 每一个变化条目可以包含多个项,每一个项对应一个变化.
  - 每一个项从新的一行开始
  - 每一个项以`-`开头  
  
 例如一个加盖时间戳的变化条目:
 ```
 %changelog
* Tue May 31 2016 Adam Miller <maxamillion@fedoraproject.org> - 0.1-1
- First bello package
- Example second item in the changelog for version-release 0.1-1
 ```
 
到此,我们终于写好了一个完整的spec文件:
{% codeblock %}
Name:           bello
Version:        0.1
Release:        1%{?dist}
Summary:        Hello World example implemented in bash script

License:        GPLv3+
URL:            https://www.example.com/%{name}
Source0:        https://www.example.com/%{name}/releases/%{name}-%{version}.tar.gz

Requires:       bash

BuildArch:      noarch

%description
The long-tail description for our Hello World Example implemented in
bash script.

%prep
%setup -q

%build

%install

mkdir -p %{buildroot}/%{_bindir}

install -m 0755 %{name} %{buildroot}/%{_bindir}/%{name}

%files
%license LICENSE
%{_bindir}/%{name}

%changelog
* Thu Jun 27 2019 Adam Miller <maxamillion@fedoraproject.org> - 0.1-1
- First bello package
- Example second item in the changelog for version-release 0.1-1
{% endcodeblock %}  

注意，我们在例子中`Source0`条目为:
```
Source0:        https://www.example.com/%{name}/releases/%{name}-%{version}.tar.gz
```
这会从我们的发布地址去拉取源码`tar.gz`包，但是我们实际上已经把bello打好包并放入了`rpmbuild/SOURCES`文件下了，因此我们需要这样指定即可:
```
Source0:        %{name}-%{version}.tar.gz
```

##### C程序cello.spec

{% codeblock %}
Name:           cello
Version:        1.0
Release:        1%{?dist}
Summary:        Hello World example implemented in C

License:        GPLv3+
URL:            https://www.example.com/%{name}
Source0:        https://www.example.com/%{name}/releases/%{name}-%{version}.tar.gz

BuildRequires:  gcc
BuildRequires:  make

%description
The long-tail description for our Hello World Example implemented in
C.

%prep
%setup -q

%build
make %{?_smp_mflags}

%install
%make_install

%files
%license LICENSE
%{_bindir}/%{name}

%changelog
* Tue May 31 2016 Adam Miller <maxamillion@fedoraproject.org> - 1.0-1
- First cello package

{% endcodeblock %}

##### Java项目jello.spec

##### 前端静态页面+nginx hello.spec文件

#### 构建RPM包
##### 源码形式的SRPM构建
- 保留部署到环境的RPM的某个名称版本发布的确切来源。 这包括确切的SPEC文件，源代码和所有相关补丁。 这对于回顾历史和调试很有用。
- 能够在不同的硬件平台或体系结构上构建二进制RPM。
```
rpmbuild -bs bello.spec
```

##### 二进制形式的RPM构建
###### 从SRPM构建
```
rpmbuild --rebuild ~/rpmbuild/SRPMS/bello-0.1-1.el7.src.rpm
```
###### 直接从spec文件构建
```
rpmbuild -bb bello.spec
```

### 引用
- [RPM Packaging Guide](https://rpm-packaging-guide.github.io/)
- [creating-rpm-packages](https://docs.fedoraproject.org/en-US/quick-docs/creating-rpm-packages/index.html)