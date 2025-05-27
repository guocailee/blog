---
{"dg-publish":true,"permalink":"/Program/Build Tools/Gradle 7.0 依赖统一管理的全新方式/","noteIcon":"","created":"2024-05-22T16:17:54.135+08:00"}
---



### 前言

随着项目的不断发展，项目中的依赖也越来越多，有时可能会有几百个，这个时候对项目依赖做一个统一的管理很有必要，我们一般会有以下需求:

1.  项目依赖统一管理，在单独文件中配置
2.  不同 Module 中的依赖版本号统一
3.  不同项目中的依赖版本号统一

针对这些需求，目前其实已经有了一些方案:

1.  使用循环优化 Gradle 依赖管理
2.  使用 buildSrc 管理 Gradle 依赖
3.  使用 includeBuild 统一配置依赖版本

上面的方案支持在不同 Module 间统一版本号，同时如果需要在项目间共享，也可以做成 Gradle 插件发布到远端，已经基本可以满足我们的需求。

不过 Gradle 7.0 推出了一个新的特性，使用 Catalog 统一依赖版本，它支持以下特性:

1.  对所有 module 可见，可统一管理所有 module 的依赖
2.  支持声明依赖 bundles，即总是一起使用的依赖可以组合在一起
3.  支持版本号与依赖名分离，可以在多个依赖间共享版本号
4.  支持在单独的 libs.versions.toml 文件中配置依赖
5.  支持在项目间共享依赖

### 使用 Version Catalog

注意，Catalog 仍然是一个孵化中的特性，如需使用，需要在 settings.gradle 中添加以下内容:

```javascript
enableFeaturePreview('VERSION_CATALOGS')
```

从命名上也可以看出，Version Catalog 其实就是一个版本的目录，我们可以从目录中选出我们需要的依赖使用。

我们可以通过如下方式使用 Catalog 中声明的依赖:

```properties
dependencies {
    implementation(libs.retrofit)
    implementation(libs.groovy.core)
}
```

在这种情况下，libs 是一个目录，retrofit 表示该目录中可用的依赖项。与直接在构建脚本中声明依赖项相比，Version Catalog 具有许多优点:

-   对于每个 catalog，Gradle 都会生成类型安全的访问器，以便您在 IDE 中可以自动补全。(注: 目前在 build.gradle 中还不能自动补全，可能是指 kts 或者开发中？)
-   声明在 catalog 中的依赖对所有 module 可见，当修改版本号时，可以统一管理统一修改
-   catalog 支持声明一个依赖 bundles，即一些总是一起使用的依赖的组合
-   catalog 支持版本号与依赖名分离，可以在多个依赖间共享版本号


### 声明 Version Catalog

Version Catalog 可以在 settings.gradle(.kts) 文件中声明。

```bash
dependencyResolutionManagement {
    versionCatalogs {
        libs {
            alias('retrofit').to('com.squareup.retrofit2:retrofit:2.9.0')
            alias('groovy-core').to('org.codehaus.groovy:groovy:3.0.5')
            alias('groovy-json').to('org.codehaus.groovy:groovy-json:3.0.5')
            alias('groovy-nio').to('org.codehaus.groovy:groovy-nio:3.0.5')
            alias('commons-lang3').to('org.apache.commons', 'commons-lang3').version {
                strictly '[3.8, 4.0['
                prefer '3.9'
            }
        }
    }
}
```

别名必须由一系列以破折号 (-，推荐)、下划线 (\_) 或点 (.) 分隔的标识符组成。

标识符本身必须由 ascii 字符组成，最好是小写，最后是数字。

值得注意的是，groovy-core 会被映射成 libs.groovy.core。

如果您想避免映射可以使用大小写来区分，比如 groovyCore 会被处理成 libs.groovyCore。

### 具有相同版本号的依赖

在上面的示例中，我们可以看到三个 groovy 依赖具有相同的版本号，我们可以把它们统一起来:

```bash
dependencyResolutionManagement {
    versionCatalogs {
        libs {
            version('groovy', '3.0.5')
            version('compilesdk', '30')
            version('targetsdk', '30')
            alias('groovy-core').to('org.codehaus.groovy', 'groovy').versionRef('groovy')
            alias('groovy-json').to('org.codehaus.groovy', 'groovy-json').versionRef('groovy')
            alias('groovy-nio').to('org.codehaus.groovy', 'groovy-nio').versionRef('groovy')
            alias('commons-lang3').to('org.apache.commons', 'commons-lang3').version {
                strictly '[3.8, 4.0['
                prefer '3.9'
            }
        }
    }
}
```

除了在依赖中，我们同样可以在 build.gradle 中获取版本，比如可以用来指定 compileSdk 等:

```properties
android {
    compileSdk libs.versions.compilesdk.get().toInteger()


    defaultConfig {
        applicationId "com.zj.gradlecatalog"
        minSdk 21
        targetSdk libs.versions.targetsdk.get().toInteger()
    }
}
```

如上，可以使用 catalog 统一 compileSdk，targetSdk，minSdk 的版本号。


### 依赖 bundles

因为在不同的项目中经常系统地一起使用某些依赖项，所以 Catalog 提供了 bundle (依赖包) 的概念。依赖包基本上是几个依赖项打包的别名。

例如，您可以这样使用一个依赖包，而不是像上面那样声明 3 个单独的依赖项:

```nginx
dependencies {
    implementation libs.bundles.groovy
}
```

groovy 依赖包声明如下:

```bash
dependencyResolutionManagement {
    versionCatalogs {
        libs {
            version('groovy', '3.0.5')
            version('checkstyle', '8.37')
            alias('groovy-core').to('org.codehaus.groovy', 'groovy').versionRef('groovy')
            alias('groovy-json').to('org.codehaus.groovy', 'groovy-json').versionRef('groovy')
            alias('groovy-nio').to('org.codehaus.groovy', 'groovy-nio').versionRef('groovy')
            alias('commons-lang3').to('org.apache.commons', 'commons-lang3').version {
                strictly '[3.8, 4.0['
                prefer '3.9'
            }
            bundle('groovy', ['groovy-core', 'groovy-json', 'groovy-nio'])
        }
    }
}
```

如上所示: 添加 groovy 依赖包等同于添加依赖包下的所有依赖项。

**插件版本**

除了 Library 之外，Catalog 还支持声明插件版本。  

因为 library 由它们的 group、artifact 和 version 表示，但 Gradle 插件仅由它们的 id 和 version 标识。

因此，插件需要单独声明:

```bash
dependencyResolutionManagement {
    versionCatalogs {
        libs {
            alias('jmh').toPluginId('me.champeau.jmh').version('0.6.5')
        }
    }
}
```

然后可以在 plugins 块下面使用:

```cs
plugins {
    id 'java-library'
    id 'checkstyle'
    
    alias(libs.plugins.jmh)
}
```

### 在单独文件中配置 Catalog

除了在 settings.gradle 中声明 Catalog 外，也可以通过一个单独的文件来配置 Catalog。

如果在根构建的 gradle 目录中找到了 libs.versions.toml 文件，则将使用该文件的内容自动声明一个 Catalog。

TOML 文件主要由 4 个部分组成:

-   ```versions``` 部分用于声明可以被依赖项引用的版本
-   ```libraries``` 部分用于声明 Library 的别名
-   ```bundles``` 部分用于声明依赖包
-   ```plugins``` 部分用于声明插件

如下所示:

```ini
[versions]
groovy = "3.0.5"
checkstyle = "8.37"
compilesdk = "30"
targetsdk = "30"

[libraries]
retrofit = "com.squareup.retrofit2:retrofit:2.9.0"
groovy-core = { module = "org.codehaus.groovy:groovy", version.ref = "groovy" }
groovy-json = { module = "org.codehaus.groovy:groovy-json", version.ref = "groovy" }
groovy-nio = { module = "org.codehaus.groovy:groovy-nio", version.ref = "groovy" }
commons-lang3 = { group = "org.apache.commons", name = "commons-lang3", version = { strictly = "[3.8, 4.0[", prefer="3.9" } }

[bundles]
groovy = ["groovy-core", "groovy-json", "groovy-nio"]

[plugins]
jmh = { id = "me.champeau.jmh", version = "0.6.5" }
```

如上所示，依赖可以定义成一个字符串，也可以将 module 与 version 分离开来。

其中 versions 可以定义成一个字符串，也可以定义成一个范围，详情可参见 rich-version:

```ini
[versions]
my-lib = { strictly = "[1.0, 2.0[", prefer = "1.2" }
```

### 在项目间共享 Catalog

Catalog 不仅可以在项目内统一管理依赖，同样可以实现在项目间共享。

例如我们需要在团队内制定一个依赖规范，不同组的不同项目需要共享这些依赖，这是个很常见的需求。

### 通过文件共享

Catalog 支持通过从 Toml 文件引入依赖，这就让我们可以通过指定文件路径来实现共享依赖。

如下所示，我们在 settins.gradle 中配置如下:

```properties
dependencyResolutionManagement {
    versionCatalogs {
        libs {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}
```

此技术可用于声明来自不同文件的多个目录:

```python
dependencyResolutionManagement {
    versionCatalogs {
        // 声明一个'testLibs'目录, 从'test-libs.versions.toml'文件中
        testLibs {
            from(files('gradle/test-libs.versions.toml'))
        }
    }
}
```

### 发布插件实现共享

虽然从本地文件导入 Catalog 很方便，但它并没有解决在组织或外部消费者中共享 Catalog 的问题。

我们还可能通过 Catalog 插件来发布目录，这样用户直接引入这个插件即可。

Gradle 提供了一个 Catalog 插件，它提供了声明然后发布 Catalog 的能力。

### 首先引入两个插件

```nginx
plugins {
    id 'version-catalog'
    id 'maven-publish'
}
```

然后，此插件将公开可用于声明目录的 catalog 扩展。

**定义目录**

上面引入插件后，即可使用 catalog 扩展定义目录。

```javascript
catalog {
    
    versionCatalog {
        from files('../libs.versions.toml')
    }
}
```

然后可以通过 maven-publish 插件来发布目录。

### 发布目录

```properties
publishing {
    publications {
        maven(MavenPublication) {
            groupId = 'com.zj.catalog'
            artifactId = 'catalog'
            version = '1.0.0'
            from components.versionCatalog
        }
    }
}
```

我们定义好 groupId，artifactId，version，from 就可以发布了。

我们这里发布到 mavenLocal，您也可以根据需要配置发布到自己的 maven

以上发布的所有代码可见: Catalog 发布相关代码

### 使用目录

因为我们已经发布到了 mavenLocal，在仓库中引入 mavenLocal 就可以使用插件了。

```cs
# settings.gradle
dependencyResolutionManagement {
    
    repositories {
        mavenLocal()
        
    }
}

enableFeaturePreview('VERSION_CATALOGS')
dependencyResolutionManagement {
    versionCatalogs {
        libs {
            from("com.zj.catalog:catalog:1.0.0")
            
            version("groovy", "3.0.6")
        }
    }
}
```

如上就成功引入了插件，可以使用 catalog 中的依赖了。

这样就完成了依赖的项目间共享，以上使用的所有代码可见: Catalog 使用相关代码。

### 总结

项目间共享依赖是比较常见的需求，虽然我们也可以通过自定义插件实现，但还是不够方便。

Gradle 官方终于推出了 Catalog，让我们可以方便地实现依赖的共享，Catalog 主要具有以下特性:

1.  对所有 module 可见，可统一管理所有 module 的依赖
2.  支持声明依赖 bundles，即总是一起使用的依赖可以组合在一起
3.  支持版本号与依赖名分离，可以在多个依赖间共享版本号
4.  支持在单独的 libs.versions.toml 文件中配置依赖
5.  支持在项目间共享依赖

 [https://mp.weixin.qq.com/s/4jhimAEHRtQa4U1BCYk4Pg](https://mp.weixin.qq.com/s/4jhimAEHRtQa4U1BCYk4Pg)
