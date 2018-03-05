## Go依赖管理机制

无论何种语言，依赖管理都是一个比较复杂的问题。而Go语言中的依赖管理机制目前还是让人比较失望的。在1.6版本之前，官方只有把依赖放在GOPATH中，并没有多版本管理机制；1.6版本（1.5版本是experimental feature）引入vendor机制，是包依赖管理对一次重要尝试。他在Go生态系统中依然是一个热门的争论话题，还没有想到完美的解决方案。

### 1 看其它

我们先来看看其它语言怎么解决，例举两种典型的管理方式：

#### 1.1 Java

开发态，可以通过maven和gradle工具编辑依赖清单列表/脚本，指定依赖库的位置/版本等信息，这些可以帮助你在合适的时间将项目固化到一个可随时随地重复编译发布的状态。这些工具对我来说已经足够优雅有效。但maven中也有不同依赖库的内部依赖版本冲突等令人心烦的问题。尤其是在大型项目中的依赖传递问题，若团队成员对maven机制没有足够了解下，依赖scope的滥用，会让整个项目工程的依赖树变得特别的巨大而每次编译效率低下。运行态，目前Java也没有很好的依赖管理机制，虽有classloader可以做一定的隔离，但像OSGi那种严格的版本管理，会让使用者陷入多版本相互冲突的泥潭。

#### 1.2 Node.js

npm是Node.js的首选模块依赖管理工具。npm通过一个当前目录的 package.json 文件来描述模块的依赖，在这个文件里你可以定义你的应用名称( name )、应用描述( description )、关键字( keywords )、版本号( version )等。npm会下载当前项目依赖模块到你项目中的一个叫做node_modules的文件夹内。与maven/gradle不同的是，maven最终会分析依赖树，把相同的软件默认扁平化取最高版本。而npm支持nested dependency tree。nested dependency tree是每个模块依赖自己目录下node_modules中的模块，这样能避免了依赖冲突, 但耗费了更多的空间和时间。由于Javascript是源码发布，所以开发态与运行态的依赖都是基于npm，优先从自己的node_modules搜索依赖的模块。

#### 2 go get

Go对包管理一定有自己的理解。对于包的获取，就是用go get命令从远程代码库(GitHub, Bitbucket, Google Code, Launchpad)拉取。并且它支持根据import package分析来递归拉取。这样做的好处是，直接跳过了包管理中央库的的约束，让代码的拉取直接基于版本控制库，大家的协作管理都是基于这个版本依赖库来互动。细体会下，发现这种设计的好处是去掉冗余，直接复用最基本的代码基础设施。Go这么干很大程度上减轻了开发者对包管理的复杂概念的理解负担，设计的很巧妙。

当然，go get命令，仍然过于简单。对于现实过程中的开发者来说，仍然有其痛苦的地方：

- 缺乏明确显示的版本。团队开发不同的项目容易导入不一样的版本，每次都是get最新的代码。尤其像我司对开源软件管理非常严格，开源申请几乎是无法实施。
- 第三方包没有内容安全审计，获取最新的代码很容易引入代码新的Bug，后续运行时出了Bug需要解决，也无法版本跟踪管理。
- 依赖的完整性无法校验，基于域名的package名称，域名变化或子路径变化，都会导致无法正常下载依赖。我们在使用过程，发现还是有不少间接依赖包的名称已失效了（不存在，或又fork成新的项目，旧的已不存维护更新）。

而Go官方对于此类问题的建议是把外部依赖的代码复制到你的源码库中管理。把第三方代码引入自己的代码库仍然是一种折中的办法，对于像我司的软件开发流程来说，是不现实的：

- 开源扫描会扫描出是相似的代码时，若License不是宽松的，则涉及到法律风险，若是宽松的，开源扫描认证确认工作也很繁琐。
- 如何升级版本，代码复制过来之后，源始的项目的代码可以变化很大了，无明显的版本校验，借助工具或脚本来升级也会带来工作量很大。
- 复制的那一份代码已经开始变成私有，第三方代码的Bug只能自己解决，难以贡献代码来修复Bug，或通过推动社区来解决。
- 普通的程序问题可能不是很大问题，最多就是编译时的依赖。但如果你写的是一个给其他人使用的lib库，引入这个库就会带来麻烦了。你这个库被多人引用，如何管理你这个库的代码依赖呢？

好在开源的力量就是大，Go官方没有想清楚的版本管理问题，社区就会有人来解决，我们已经可以找到许多不错的解决方案，不妨先参考下[官方建议](https://github.com/golang/go/wiki/PackageManagementTools)。

### 3 vendor

vendor是1.5引入为体验，1.6中正式发布的依赖管理特性。Go团队在推出vendor前已经在Golang-dev group上做了长时间的调研。最终Russ Cox在Keith Rarick的proposal的基础上做了改良，形成了Go 1.5中的vendor:

- 不rewrite gopath
- go tool来解决
- go get兼容
- 可reproduce building process

并给出了vendor机制的”4行”诠释：

    If there is a source directory d/vendor, then, when compiling a source file within the subtree rooted at d, import “p” is interpreted as import “d/vendor/p” if that exists.
    
    When there are multiple possible resolutions,the most specific (longest) path wins.
    
    The short form must always be used: no import path can contain “/vendor/” explicitly.
    
    Import comments are ignored in vendored packages.

总结解释起来：

- vendor是一个特殊的目录，在应用的源码目录下，go doc工具会忽略它。
- vendor机制支持嵌套vendor，vendor中的第三方包中也可以包含vendor目录。
- 若不同层次的vendor下存在相同的package，编译查找路径优先搜索当前pakcage下的vendor是否存在，若没有再向parent pacakge下的vendor搜索（x/y/z作为parentpath输入，搜索路径：x/y/z/vendor/path->x/y/vendor/path->x/vendor/path->vendor/path)
- 在使用时不用理会vendor这个路径的存在，该怎么import包就怎么import，不要出现import “d/vendor/p”的情况。vendor是由go tool隐式处理的。
- 不会校验vendor中package的import path是否与canonical import路径是否一致了。

vendor机制看似像node.js的node_modules，支持嵌套vendor，若一个工程中在着两个版本的相的包，可以放在不同的层次的vendor下：

- 优点：可能解决不同的版本依赖冲突问题，不同的层次的vendor存放在不同的依赖包。
- 缺点：由于go的package是以路径组织的，在编译时，不同层次的vendor中相同的包会编译两次，链接两份，程序文件变大，运行期是执行不同的代码逻辑。会导致一些问题，如果在package init中全局初始化，可能重复初化出问题，也可能初化为不同的变量（内存中不同），无法共享获取。像之前我们遇到gprc类似的问题就是不同层次的相同package重复init导致的，见社区反馈。

所以Russ Cox期望大家良好设计工程布局，作为lib的包不携带vendor更佳 ，一个project内的所有vendor都集中在顶层vendor里面。


### 4 后续

Go的包依赖问题依旧困扰着开发人员，嵌套vendor可以一定程度解决多版本的依赖冲突问题，但也引入多份编译导致的问题。目前社区也在一直讨论如何更好的解决，将进入下一个改进周期。这次将在Peter Bourgon的主持下正式启动：go packaging proposal process，当前1.8版本特性已冻结，不知这个改进是否会引入到1.9版本中。

**原文：[http://lanlingzi.cn/post/technical/2016/1120_go_deps_mgnt/](http://lanlingzi.cn/post/technical/2016/1120_go_deps_mgnt/)**














