Ansible：像系统管理员一样思考的自动化框架
======

这些年来，我已经写了许多关于DevOps工具的文章，也培训了这方面的人员，尽管这些工具很棒，但很明显，大多数都是按照开发人员的思路设计出来的。这也没有什么问题，因为以编程的方式接近配置管理是重点。不过，直到我开始接触Ansible，我才觉得这才是系统管理员喜欢的东西。

喜欢的一部分原因是Ansible与客户端计算机通信的方式，是通过SSH的。作为系统管理员，你们都非常熟悉通过SSH连接到计算机，所以从单词“去”的角度来看，你对Ansible的了解要比其他选择更好。

考虑到这一点，我打算写一些文章，探讨如何使用Ansible。这是一个很好的系统，但是当我第一次接触到这个系统的时候，不知道如何开始。也不是学习曲线陡峭。事实上，问题是在开始使用Ansible之前，我并没有太多的东西要学，这才是让人感到困惑的。例如，如果您不必安装客户端程序（Ansible没有在客户端计算机上安装任何软件），那么您将如何启动？

### 踏出第一步

起初Ansible对我来说非常困难的原因在于配置服务器/客户端的关系是非常灵活的，我不知道我该从何入手。事实是，Ansible并不关心你如何设置SSH系统。它会利用你现有的任何配置。需要考虑以下几件事情：

1. Ansible需要通过SSH连接到客户端计算机。
2. 连接后，Ansible需要提升权限才能配置系统，安装软件包等等。

不幸的是，这两个考虑真的带来了一堆蠕虫。连接到远程计算机并提升权限是一件可怕的事情。出于某种原因，当您只需在远程计算机上安装代理并使用Chef或Puppet处理特权升级问题时，感觉就不那么差了。 Ansible并非不安全，而是安全的决定权在你手中。

接下来，我将列出一系列潜在的配置，以及每个配置的优缺点。这不是一个详尽的清单，但是你会受到正确的启发，去思考在你自己的环境中什么是理想的配置。也需要注意，我不会提到像Vagrant这样的系统，因为尽管Vagrant在构建测试和开发的敏捷架构时非常棒，但是和一堆服务器是非常不同的，因此考虑因素是极不相似的。

### 一些SSH场景

1）在Ansible配置中，以root用户密码进入远程计算机。

拥有这个想法是一个非常可怕的开始。这个设置的“优点”是它消除了对特权提升的需要，并且远程服务器上不需要其他用户帐户。 但是，这种便利的成本是不值得的。 首先，大多数系统不会让你在不改变默认配置的情况下以root身份进行SSH登录。 由于默认的配置都在固定的位置，坦率地说，允许root用户远程连接是一个不好的主意。 其次，将root密码放在Ansible机器上的纯文本配置文件中是不合适的。 真的，我提到了这种可能性，因为这是可能的，但这是应该避免的。 请记住，Ansible允许你自己配置连接，它可以让你做真正愚蠢的事情。 但是请不要这么做。

2）使用存储在Ansible配置中的密码，以普通用户的身份进入远程计算机。

这种情况的一个优点是它不需要太多的客户端配置。 大多数用户默认情况下都可以使用SSH，因此Ansible应该能够使用凭据并且能够正常登录。 我个人不喜欢在配置文件中以纯文本形式存储密码，但至少不是root密码。 如果您使用此方法，请务必考虑远程服务器上的权限提升方式。 我知道我还没有谈到权限提升，但是如果你在配置文件中配置了一个密码，这个密码可能会被用来获得sudo访问权限。 因此，一旦发生泄露，您不仅已经泄露了远程用户的帐户，还可能泄露整个系统。

3）以普通用户身份进入远程计算机，使用具有空密码的密钥对进行身份验证。

这消除了将密码存储在配置文件中的弊端，至少在登录的过程中消除了。 没有密码的密钥对并不理想，但这是我经常做的事情。 在我的个人内部网络中，我通常使用没有密码的密钥对来自动执行许多事情，如需要身份验证的定时任务。 这不是最安全的选择，因为私钥泄露意味着可以无限制地访问远程用户的帐户，但是相对于在配置文件中存储密码我更喜欢这种方式。

4）以普通用户的身份通过SSH连接到远程计算机，使用通过密码保护的密钥对进行身份验证。

这是处理远程访问的一种非常安全的方式，因为它需要两种不同的身份验证因素来解密：私钥和密码。 如果你只是以交互方式运行Ansible，这可能是理想的设置。 当你运行命令时，Ansible会提示你输入私钥，然后使用密钥对登录到远程系统。 是的，只需使用标准密码登录并且不用在配置文件中指定密码即可完成，但是如果不管怎样都要在命令行上输入密码，那为什么不在保护层添加密钥对呢？

5）使用密码保护密钥对进行SSH连接，但是使用ssh-agent“解锁”私钥。

这并不能完美地解决无人值守，自动化的Ansible命令的问题，但是它确实也使安全设置变得相当方便。 ssh-agent程序一次验证密码，然后使用该验证进行后续连接。当我使用Ansible时，这是我想要做的事情。如果我是完全值得信任的，我通常仍然使用没有密码的密钥对，但是这通常是因为我在我的家庭服务器上工作，是不是容易受到攻击的。

在配置SSH环境时还要记住一些其他注意事项。 也许你可以限制Ansible用户（通常是你的本地用户名），以便它只能从一个特定的IP地址登录。 也许您的Ansible服务器可以位于不同的子网中，位于强大的防火墙之后，因此其私钥更难以远程访问。 也许Ansible服务器本身没有安装SSH服务器，所以根本没法访问。 同样，Ansible的优势之一是它使用SSH协议进行通信，而且这是一个协议，你有多年的时间把你的系统调整到最适合你的环境的效果。 我不是宣传“最佳实践”的忠实粉丝，因为实际上最好的做法是考虑你的环境，并选择最适合你情况的设置。

### 权限提升

一旦您的Ansible服务器通过SSH连接到它的客户端，就需要能够提升特权。 如果你选择了上面的选项1，那么你已经是root了，这是一个有争议的问题。 但是由于没有人选择选项1（对吧？），您需要考虑客户端计算机上的普通用户如何获得访问权限。 Ansible支持各种权限提升的系统，但在Linux中，最常用的选项是sudo和su。 和SSH一样，有几种情况需要考虑，虽然肯定还有其他选择。

1）使用su提升权限。

对于RedHat/CentOS用户来说，可能默认是使用su来获得系统访问权限。 默认情况下，这些系统在安装过程中配置了root密码，要想获得特殊访问权限，您需要输入该密码。使用su的问题在于，虽说它可以让您完全访问远程系统，不过您确实可以完全访问远程系统。 （是的，这是讽刺。）另外，su程序没有使用密钥对进行身份验证的能力，所以密码必须以交互方式输入或存储在配置文件中。 由于它实际上是root密码，因此将其存储在配置文件中听起来像一个可怕的想法，因为它就是。

2）使用sudo提升权限。

这就是Debian/Ubuntu系统的配置方式。 正常用户组中的用户可以使用sudo命令并使用root权限执行该命令。 随之而来的是，这仍然存在密码存储或交互式输入的问题。 由于在配置文件中存储用户的密码看起来不太可怕，我猜这是使用su的一个步骤，但是如果密码被泄露，仍然可以完全访问系统。 （毕竟，输入`sudo`和`su -`都将允许用户成为root用户，就像拥有root密码一样。）

3） 使用sudo提升权限，并在sudoers文件中配置NOPASSWD。

再者，在我的本地环境中，我就是这么做的。 这并不完美，因为它给予用户帐户无限制的root权限，并且不需要任何密码。 但是，当我这样做并且使用没有密码短语的SSH密钥对时，我可以让Ansible命令更轻松的自动化。 我会再次注意到，虽然这很方便，但这不是一个非常安全的想法。

4）使用sudo提升权限，并在特定的可执行文件上配置NOPASSWD。

这个想法可能是安全性和便利性的最佳折衷。 基本上，如果你知道你打算用Ansible做什么，那么你可以为远程用户使用的那些应用程序提供NOPASSWD权限。 这可能会让人有些困惑，因为Ansible使用Python来处理很多事情，但是有足够的尝试和错误，你应该能够弄清原理。 这是额外的工作，但确实消除了一些明显的安全漏洞。

### 计划实施

一旦你决定如何处理Ansible认证和权限提升，就需要设置它。 在熟悉Ansible之后，您可能会使用该工具来帮助“引导”新客户端，但首先手动配置客户端非常重要，以便您知道发生了什么事情。 将你熟悉的事情变得自动化比从自动化开始要好。

我已经写过关于SSH密钥对的文章，网上有无数的设置类的文章。 来自Ansible服务器的简短版本看起来像这样：

```
# ssh-keygen
# ssh-copy-id -i .ssh/id_dsa.pub remoteuser@remote.computer.ip
# ssh remoteuser@remote.computer.ip
```

如果您在创建密钥对时选择不使用密码，最后一步您应该可以直接进入远程计算机，而不用输入密码或密钥串。

为了在sudo中设置权限提升，您需要编辑sudoers文件。 你不应该直接编辑文件，而是使用：

```
# sudo visudo
```

这将打开sudoers文件并允许您安全地进行更改（保存时会进行错误检查，所以您不会意外地因为输入错误将自己锁住）。 这个文件中有一些例子，所以你应该能够弄清楚如何分配你想要的确切的权限。

一旦配置完成，您应该在使用Ansible之前进行手动测试。 尝试SSH到远程客户端，然后尝试使用您选择的任何方法提升权限。 一旦你确认配置的这种方式可以连接，就可以安装Ansible了。

### Ansible安装

由于Ansible程序仅安装在一台计算机上，因此开始并不是一件繁重的工作。 Red Hat/Ubuntu系统的软件包安装有点不同，但都不是很困难。

在Red Hat/CentOS中，首先启用EPEL库：

```
sudo yum install epel-release
```

然后安装Ansible：

```
sudo yum install ansible
```

在Ubuntu中，首先启用Ansible PPA：

```
sudo apt-add-repository spa:ansible/ansible
(press ENTER to access the key and add the repo)
```

然后安装Ansible：

```
sudo apt-get update
sudo apt-get install ansible
```

### Ansible主机文件配置

Ansible系统无法知道您希望它控制哪个客户端，除非您给它一个计算机列表。 该列表非常简单，看起来像这样：

```
# file /etc/ansible/hosts

[webservers]
blogserver ansible_host=192.168.1.5
wikiserver ansible_host=192.168.1.10

[dbservers]
mysql_1 ansible_host=192.168.1.22
pgsql_1 ansible_host=192.168.1.23
```

括号内的部分是指定组。 单个主机可以列在多个组中，而Ansible可以指向单个主机或组。 这也是配置文件，比如纯文本密码的东西将被存储，如果这是你计划的那种设置。 配置文件中的每一行配置一个主机地址，并且可以在ansible_host语句之后添加多个声明。 一些有用的选项是：

```
ansible_ssh_pass
ansible_become
ansible_become_method
ansible_become_user
ansible_become_pass
```

### Ansible Vault保险库

（译者注：Vault作为 ansible 的一项新功能可将例如passwords,keys等敏感数据文件进行加密,而非明文存放）

我也应该注意到，尽管安装程序比较复杂，而且不是在您首次进入Ansible世界时可能会做的事情，但该程序确实提供了一种方法来加密保险库中的密码。 一旦您熟悉Ansible，并且希望将其投入生产，将这些密码存储在加密的Ansible库中是非常理想的。 但是本着先学会爬再学会走的精神，我建议首先在非生产环境下使用无密码方法。

### 系统测试

最后，你应该测试你的系统，以确保客户端可以正常连接。 ping测试将确保Ansible计算机可以ping每个主机：

```
ansible -m ping all
```

运行后，如果ping成功，您应该看到每个定义的主机显示ping的消息：pong。 这实际上并没有测试认证，只是测试网络连接。 试试这个来测试你的认证：

```
ansible -m shell -a 'uptime' webservers
```

您应该可以看到webservers组中每个主机的运行时间命令的结果。

在后续文章中，我计划开始深入Ansible管理远程计算机的功能。 我将介绍各种模块，以及如何使用ad-hoc模式来完成一些按键操作，这些操作在命令行上单独处理都需要很长时间。 如果您没有从上面的示例Ansible命令中获得预期的结果，请花些时间确保身份验证正在运行。 如果遇到困难，请查阅[Ansible文档][1]获取更多帮助。

--------------------------------------------------------------------------------

via: http://www.linuxjournal.com/content/ansible-automation-framework-thinks-sysadmin

作者：[Shawn Powers][a]
译者：[Flowsnow](https://github.com/Flowsnow)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创编译，[Linux中国](https://linux.cn/) 荣誉推出

[a]:http://www.linuxjournal.com/users/shawn-powers
[1]:http://docs.ansible.com