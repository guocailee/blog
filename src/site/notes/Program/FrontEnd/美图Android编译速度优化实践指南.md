---
{"dg-publish":true,"permalink":"/Program/FrontEnd/美图Android编译速度优化实践指南/","noteIcon":"","created":"2025-03-06T21:28:25.975+08:00"}
---


**导读：** 本文的主题是美图秀秀的 Android 编译速度实践指南，通过对本文整体方案的演进的阅读，读者也可以把方案快速落地到自己公司的产品中；首先会和大家介绍一下做编译加速优化的背景，以及我们在做优化过程中，每一期是如何演进升级的，以及我们最终在开发上形成的约束和规范，最终谈一谈未来那些方向我们可以继续深挖，保持持续更新和迭代。


## 01 背景介绍

由于美图秀秀 Android 工程庞大，优化前全量编译 **8m45.2s**、增量编译**2m35.8s**，造成在开发期间研发同学需长时间等待编译构建完成，严重影响研发效率，因此针对工程进行编译优化势在必行。

目前优化后全量编译（耗时 **1m49.8s**、降幅达 **79.1%**），增量编译（耗时 **40.6s**、降幅达 **73.9%**）。

![](/img/user/z-attchements/media/640-21.png)

编译加速优化 1 期包含 jvm 优化、aspectjx 和 firebase 任务屏蔽、res 精简，编译加速优化 2 期包含 AGP 升级和模块 aar 发布；编译加速优化 3 期包含动态版本依赖、自动发布以及模块安全隔离；编译加速优化 4 期包含静态版本、动态计算、依赖查询和配置切换自动化；通过上述 1-4 期优化，我们可以做到和日常开发流程一样，不用关注各种模块版本号、源码和 aar 之类，全局自动检测和适配。该方案已经在美图公司多数产品线 Android 项目上落地，为此希望借此机会，和各位交流美图秀秀 Android 的整体编译优化方案。


## 02 优化演进

**1.\*\***编译优化第 1 期 \*\*

![](/img/user/z-attchements/media/640-21.png)

编译优化第 1 期通过屏蔽 aspectjx、lint、测试相关任务、jvm 内存优化、resConfigs 精减等手段，增量编译整体耗时降低**22.4%，**但整体降幅有限，因此也为我们后续做二期优化埋下伏笔。

**（1）Task 任务分析**

![](/img/user/z-attchements/media/640-26.png)

通过上述任务数据截图，我们知道类似 aspectjx 等 transforms 任务是相对比较耗时，达**30s+**，因此自然我们可以在本地开发环境下针对类似的任务进行屏蔽，从而降低编译耗时。

**（2）jvm 优化**

![](/img/user/z-attchements/media/640-28.png)

根据官方的描述，我们使用并行垃圾回收期要比 G1 垃圾回收器性能更高，所以可以-XX:+UseParallelGC 把这块配置加速。

![](/img/user/z-attchements/media/640-25.png)

在我们电脑设备硬件条件允许下，可以适当增加包括 dexOptions 中 maxProcessCount 以及线程数、jvm 内存等。

**2.\*\***编译优化第 2 期 \*\*

![](/img/user/z-attchements/media/640-25.png)

编译优化第 2 期通过把工程中所有子模块发布仓库后进行外部依赖及升级 AGP 至 4.0.2 版本等手段，增量编译速度从**2m35.8s**降低至**40.6s**，降幅达**73.9%**，全量编译速度从**8m45.2s**降低至**1m49.8s**，降幅达**79.1%，**优化效果十分明显，但由于每次都要改版本号，操作比较繁琐，因此我们在编译优化三期对应做了升级和调整。

**（1）模块内部依赖和外部依赖对比**

```javascript
// 内部依赖方式
implementation project(':app:base')
// 外部依赖方式
implementation "com.xxx.library:base:xxx"
```

上面列举说明了内部依赖和外部依赖方式，内部依赖对应的就是源码依赖，而外部依赖则是通过模块发布到仓库之后根据 group、moudle、version 进行 aar 依赖。

**（2）模块发布方案选型**

![](/img/user/z-attchements/media/640-25.png)

由于我们发布的模块需要支持不同渠道、变体依赖不同的第三方库（如推送分为华为、Vivo、Oppo、Google），因此最终选择 Google 官方最新的 publish 进行仓库管理，如上图所示，通过 components.getByName("all")，在发布模块时就可以针对不同渠道的 aar 包，保证在模块通过外部依赖加载时，程序运行正常运行，不会在某些渠道下报找不到类问题，这个版本 AGP 必须要升级至 3.6.0 + 及以上，因此还需要对 AGP 进行相应升级，官方参考链接：使用 Maven Publish 插件。

**（3）AGP 升级前后对比**

![](/img/user/z-attchements/media/640-22.png)

从 AGP 升级从 3.5.4 版本升至 4.0.2 版本，greenDao 和插件库等有可能导致会无法编译问题，因此我们针对相应的插件使用的 gradle 相应的 api 进行升级处理。

**（4）内部依赖和外部依赖冲突**

```javascript
configurations.all {     
resolutionStrategy.dependencySubstitution {         
  if (IS_BASE_AAR_MODE.toBoolean()) {
    substitute project(":app:base") because "using base aar version" with module("com.xxx.library:base:${BASE_VERSION}")
    substitute module("com.xxx.library:base") because "replace base aar version" with module("com.xxx.library:base:${BASE_VERSION}")
  } else {
    substitute module("com.xxx.library:base") because "using base project version" with project(":app:base")
  }
 }
}
```

由于我们在发布模块是采用的是内部依赖形式，而在开发过程中是通过模块外部依赖，因此在运行工程过着中容易造成内部依赖和外部依赖冲突，导致最终编译失败，因此我们可以通过上述代码，把 base 库全部替换为源码或者统一特定版本的 aar，从而解决遇到的问题，详细大家可以查阅 gradle 官方链接：Customizing Resolution。

**（5）android.support 引发血案**

![](/img/user/z-attchements/media/640-23.png)

由于美图秀秀的工程是通过官方提供 android.useAndroidX 和 android.enableJetifier 进行迁移的，而我们在进行模块化发布和外部依赖时发现找不到对应的布局相应类，造成的原因是我们自定义的布局是以 android.support 命名，对应的发布的模块已经对自定义布局类包名变更，但加载自定义布局的其他模块没有进行一并修改，为此我们在进行模块化开发时特别需要注意，不要随便以 android.support 对自定义布局相关类进行包名命名，详细大家可以查阅 androidx 官方链接：androidx 迁移。

**（6）发布方式初探**

在编译加速优化 2 期时，主要通过两种方式，一种是基于 CI 上的打包平台，在里面新增流水线执行发布任务，但会和线上打 apk 包任务混在一起，从而占用打包资源，后面我们就尝试通过 gitlab 的脚本进行发布，通过打 tag 触发对应的模块发布任务，但这种方式有两个弊端，第 1 个就是操作比较繁琐，需要先把代码 push 到 git 上，然后在发布 tag 进行发布，第 2 个就是容易对 tag 造成污染，因为目前 tag 一般用于线上封版后触发，从而记录一个版本的生命周期结束，为此我们在编译加速优化 3 期时，针对此问题进行全面的优化。

**3.\*\***编译优化第 3 期 \*\*

![](/img/user/z-attchements/media/640-25.png)

编译优化第 3 期是基于动态版本和模块自动发布、以及兼顾安全而设计，主要为了解决研发同学在开发过程中需要手动修改版本，导致版本维护混乱易出错问题；但编译优化第 3 期有个小缺陷，那就是如果研发同学在子分支开发需求时由于模块外部依赖版本也会被动态变更，导致开发中断或者编译失败，而且随着迭代的进行，一个周期内的动态版本会越来越多，那么会增加依赖库查询的时间，不过相比编译优化第 2 期已经有了质的飞跃，已经往自动化方向靠齐，因此我们在编译优化四期针对三期进行全面的迭代和升级。

**（1）模块发布版本设计**

```kotlin
def getVerSuffix() {
    String verSuffixStr = ""
    final String pipelineStr = System.getenv("CI_PIPELINE_ID")
    if (pipelineStr != null) {
        final long pipelineNo = pipelineStr.toLong()
        println("Build: pipelineNo:${pipelineStr}")
        verSuffixStr = "." + String.format("%010d", pipelineNo)
        println("Build: versionSuffix:${verSuffixStr}")
    }
    return verSuffixStr
}
```

要实现动态版本，那么就得寻找到一个永远递增且唯一的版本，而 gitlab 的流水线号恰恰满足这个条件，我们可以通过流水线号保留 10 位作为动态版本号后缀，为了做好每个迭代或者版本周期内模块版本相互隔离。

**（2）模块外部依赖示例**

```nginx
def DEV_VERSION = "1.0.0.0."
if (getProp('baseIsDependOnCode') == 'false') {
    api "com.xxx.library:base:${DEV_VERSION + '+'}"
} else {
    api project(':app:base')
}
```

如上述示例代码所示，我们的模块的发布版本由固定的版本号 + 流水线号构成，动态版本号前缀是由每个版本的发布版本号名称构成，如：1.0.0.0，这样就可以通过动态版本对模块进行动态依赖。

**（3）模块外部依赖示例**

```sql
# base发布任务.
core-deploy:
  stage: deploy
  # 发布执行命令，可以根据实际发布需要进行配置,支持多条命令.
  script:
    - ./gradlew clean --stacktrace app:base:publish
  # 限定docker runner(必须)
  tags:
    - docker
  # 保留mapping.txt、usage.txt文件, 可根据需要进行路径调整.
  artifacts:
    when: always
  rules:
    - if: ($CI_COMMIT_BRANCH =~ /^release.*/ || $CI_COMMIT_BRANCH == "develop")
      changes:
        - app/base/**/*
      when: always
      allow_failure: true
    - if: $CI_COMMIT_TAG
      when: always
      allow_failure: true
    - if: '$CI_PIPELINE_SOURCE == "push"'
      when: manual
```

如上述示例代码所示，自动发布我们主要是通过 gitlab 的脚本进行实现，如上截图所示，当代码合入 dev 及 release 相关分支时，如果检测到对应模块的文件目录有发生变更，就会触发自动发布；而通过打 tag 则会自动触发所有模块进行自动发布；通过 push 代码到 git 仓库，只能手动触发；在发布 aar 失败后，我们也可以通过手动触发发布来进行补救；相应配置大家可以参考 gitlab 官方链接：gitlab 脚本配置。

**（4）通过认证发布和依赖**

```properties
publishing {
    publications {
        maven(MavenPublication) {
            from components.getByName("all")
            artifact sourceJar
            groupId MAVEN_GROUP_ID
            artifactId rootProject.ext.moduleNames.base
            version getPubVersion(rootProject.ext.modulePaths.base)
            pom {
                packaging = "aar"
            }
        }
    }

    repositories {
        maven {
            url MAVEN_REPOSITORY_URL
            name GIT_LAB
            credentials(HttpHeaderCredentials) {
                name = JOB_NAME
                value = JOB_TOKEN
            }
            authentication {
                header(HttpHeaderAuthentication)
            }
        }
    }
}
```

如上代码所示，模块发布是通过 gitlab 的特性，即通过 JOB_TOKEN 进行安全证书后进行模块发布。

```properties
maven {
    url "http://www.xxx.com/api/v4/projects/xxx/packages/maven"
    name "GitLab"
    credentials(HttpHeaderCredentials) {
        name = 'Private-Token'
        value = 'xxx'
    }
    authentication {
        header(HttpHeaderAuthentication)
    }
    content {
        includeGroup "com.xxx.library"
    }
}
```

通过私钥对代码仓库进行访问，这种方式可以把发布的模块控制在对代码仓库至少有读写的权限才可以访问模块的 aar，从而确保整个模块的访问和依赖的安全性。

**4.\*\***编译优化第 4 期 \*\*

![](/img/user/z-attchements/media/640-27.png)

编译优化第 4 期是通过模块之间依赖是通过静态版本动态计算，并结合模块本地是否有修改、arr 对应版本仓库中是否存在，最终来决定模块是源码还是 arr 进行依赖，从而解决开发被中断以及线上动态版本越来越多导致的编译速度下降问题，更加自动化和智能化，彻底解决编译优化三期遇到的开发过程中被中断以及版本号堆叠后速度变慢问题。

**（1）模块发布版本新方案**

```ruby
def getPubVersion(String modularPath) {
    // 获取提交日期的时间戳作为版本号
    final String pubVersion = ("git log -1 --pretty=%at " + modularPath).execute().text.trim()
    return pubVersion
```

模块的版本是通过获取每个模块的最后一条记录的首次提交时间戳作（如果以 commitId 作为版本号，则代码 rebase 之后可能会发生变更）为版本号（同发布和依赖），这样可以确保模块和 arr 保持源码高度一致性。

**（2）检测模块变更**

```ruby
def isModularChange(String modularPath) {
    def changeLog = ("git status --short " + modularPath).execute().text.trim()
    if (changeLog == null || changeLog.size() == 0) {
        return false
    }
    return true
}
```

而模块代码是否发生变更，是通过 git 的特性，即 status 来判断本地是否有文件修改或者未提交的内容。

**（3）检测模块发布**

```ruby
def isModularPublish(String modularName, String versionName) {
       def pomQuery =  project.dependencies.createArtifactResolutionQuery().forModule(
            MAVEN_GROUP_ID, modularName, versionName)
            .withArtifacts(MavenModule, MavenPomArtifact)
    def pomResult = pomQuery.execute().resolvedComponents
    if (pomResult != null && pomResult.size() > 0) {
         return true
    }
    return false
}
```

通过获取到的每个模块的提交记录的时间戳作为版本号，我们就可以通过 group、name、version 去查询 maven 仓库中是否已经发布该版本，如果存在则外部依赖，不存在则内部依赖。

**（4）内外依赖动态切换 md5 校验**

```ruby
def checkoutBuildMd5() {
    def buildCachePath = rootProject.rootDir.path + "/app/build/"
    File buildCacheDir = new File(buildCachePath)
    def buildCacheMd5 = buildCacheStr.md5()
    File buildMd5File = new File(buildCacheDir, buildCacheMd5 + ".obs")
    if (buildCacheDir.exists()) {
        if (!buildMd5File.exists()) {
            delete(buildCachePath)
            buildCacheDir.mkdirs()
            buildMd5File.createNewFile()
        }
    }
}
```

我们在开发和构建中，经常遇到场景就是外部依赖和内部依赖动态切换，比如在某一个模块合入 dev 后进行自动化发布，另一个研发同学拉取 dev 分支代码，刚开始该模块是以内部依赖进行加载，待模块发布完成后，又变成外部依赖，此时由于 build 的构建缓存，如果没有进行清理，则最终会报类冲突而无法编译成功（需要手动 clean），因此我们针对类似这种场景进行优化，方案就是每次构建完成后我们把所有模块外部依赖和内部依赖的名称进行字符串拼接，最后在换算成 md5，写入到构建缓存中文件中，如果下次构建时发现两者不同，则进行自动清理，以达到不需要人工干预，自动化通过 md5 值进行校验。

**（5）简化配置**

```cs
/**
 * 是否屏蔽aspectjx开关打开
 * @return
 */
def isDisableAspectjx() {
    return getProp('isDisableAspectjx') ?: IS_DEBUG
}

/**
 * 是否开启firebase开关打开
 * @return
 */
def isDisableFirebase() {
    return getProp('isDisableFirebase') ?: IS_DEBUG
}
```

针对 debug 开发模式，默认屏蔽 aspectjx 和 firebase 的 mapping 上传任务，减少编译耗时，当然研发同学也可以通过配置自行开启对应的 task 开关，从而达到默认不需要增加任何配置，也可以自动适配编译优化方案。

## 03  开发规范

![](/img/user/z-attchements/media/640-21.png)

如上图所示，展示了整个模块化开发的流程，和编译加速优化之前的流程是一致的，减少学习和适应成本，默认情况下所有配置项在 debug 环境下都是自动开启编译加速优化方案，如果对特定业务有需求，比如想打开 aspectjx、全源码依赖、各模块依赖等，则支持 local.properties 进行相应配置。什么依赖开关、版本号、编译配置统统丢掉，我们要做的就是大道至简，全部实现自动化和智能化。

## 04  未来展望

工欲善其事，必先利其器，除了上述做了编译 4 期优化，编译加速优化之路还是有很多可以继续完善的点，比如像模块自动发布之后监控是否发布成功或者如何根据工程构建产物持续监控和优化现有的方案等。

**（1）模块发布监控**

```powershell
.notify: &notify |
    curl 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx' \
       -H 'Content-Type: application/json' \
       -d "
       {
            \"msgtype\": \"text\",
            \"text\": {
                \"content\": \"Tag: $CI_COMMIT_TAG 打包结束 \n触发者: $GITLAB_USER_EMAIL \n详情：http://www.xxx.com/maven/-/pipelines\"
            }
       }"
```

模块发布监控意义在于在模块发布成功或者失败后，直接触发通知，发送到对应的企业微信群，那么我们就可以实时监控发布的情况，来定位、解决线上模块发布问题。

**（2）工程构建监控**

**![](/img/user/z-attchements/media/640-24.png)**

通过官方提供的 scan 产物链接，我们可以非常方便的分析构建过程中可能导致的问题，通过查看日志快速定位和引导修复编译问题，核心配置代码如下所示（tips：需要在 setting.gradle 文件中配置）。

```nginx
plugins {
    id "com.gradle.enterprise" version "3.7.1"
}

gradleEnterprise {
    buildScan {
        captureTaskInputFiles = true
        uploadInBackground = false
        publishAlways()
        termsOfServiceUrl = "https://gradle.com/terms-of-service"
        termsOfServiceAgree = "yes"
    }
}
```

**参考阅读：** 

-   [基于 etcd 实现大规模服务治理应用实战](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557639&idx=1&sn=7655a93ad702670b5ff685e6aa7b7468&chksm=8139821fb64e0b09d47272952230a73325af8a9a5dbb14f94cd8c4d2c2ba6e2e5cc411555885&scene=21#wechat_redirect)
-   [深入剖析 Redis 客户端 Jedis 的特性和原理](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557624&idx=1&sn=11274fdc8454ab6ebc4ba1d562148a6c&chksm=813982e0b64e0bf671d7b3a32219e5f2a905372018d1add48f78ac0ff63b9e033a7c58b1782c&scene=21#wechat_redirect)
-   [终于！12 年后 Golang 支持泛型了！（内含 10 个实例）](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557609&idx=1&sn=9ee127addb3d59542d13d92328bef739&chksm=813982f1b64e0be77d1226bbb8ae3c610246cb03f45dbd18376a324659fa88eeb4ff5daf4e2f&scene=21#wechat_redirect)
-   [浅谈 Apache APISIX 的可观测性](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557572&idx=1&sn=2927f2304ef6f8975992d33624e13b4c&chksm=813982dcb64e0bcac3f596d3c6ffb906f0094a17168cb6694e1d6fa486d47eee03b2af509214&scene=21#wechat_redirect)
-   [百度爱番番数据分析体系的架构与实践](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653557555&idx=1&sn=adb40ef8a88d3dae1cb7d674dbf9a569&chksm=813982abb64e0bbdebab9a1a01172d752a1125e4aa8f56c43e0cd0f2f113db049fd535c148c0&scene=21#wechat_redirect)

 [https://mp.weixin.qq.com/s/u0Vx0zC73CRS7awYow0ALg](https://mp.weixin.qq.com/s/u0Vx0zC73CRS7awYow0ALg)
