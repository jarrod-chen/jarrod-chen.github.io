---
layout:     post
title:      Android Support库迁移至AndroidX指南
subtitle:   
date:       2020-08-16
author:     Jarrod
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - AndroidX
---
# Android Support库迁移至AndroidX指南
### What----AndroidX是什么?       

AndroidX是Jetpack中所有的库的软件包名称。2018年谷歌I/O 发布了一系列辅助Android开发者的实用工具，合称Jetpack，以帮助开发者构建出色的 Android 应用。Jetpack 是一套库、工具和指南，可帮助开发者更轻松地编写优质应用。这些组件可帮助您遵循最佳做法、让您摆脱编写样板代码的工作并简化复杂任务，以便您将精力集中放在业务代码上。



### Why----为什么要做这样的迁移？

有以下四点原因：

1. Android Support Library已经完成了它的任务了。28.0是Android Support命名空间的最后一个发布版本了，同时Android Support 这个命名空间在不久之后将不再进行维护。所以，如果想Support Library中的bug得到修复或者使用新的功能，就需要迁移到AndroidX上。

2. 更好的包管理。使用AndroidX，可以得到标准化和独立的版本控制，以及更标准化的命名和更频繁的更新发布。

3. 很多库已经迁移到了AndroidX上了，比如Google Play Services，Firebase，Butterknife等。

4. 所有新发布的Jetpack库都将使用AndroidX命名空间，所以如果想使用Jetpack中的新功能，你就需要迁移到AndroidX上。

   

### Prepares----迁移准备

在迁移项目代码到AndroidX上之前，最好做这些事情：

- 备份项目，因为迁移可能会修改项目中到不少文件。

- 创建一个新的分支去进行迁移工作，等迁移完成后再进行分支合并。

- 如果可以的话，在迁移过程中，尽量暂停功能的开发或者最小化的开发（至少不要在迁移过程中），这样能减少代码合并时可能发生的合并冲突。

  

### How----如何迁移

在迁移过程中，重点在于解决发生的错误，基本上错误都是项目代码及第三方库中牵扯到之前Support Library中类引用变更问题，需要更新三方库或者项目中引用的类，使项目能编译通过并且通过所有测试。

#### Step 1:更新所有的Support Library版本到28

Android官方不推荐直接从旧版本的Support Library(比如26或27)迁移到AndroidX，因为这样不仅面对命名空间修改的问题，还有可能面临因为新库与旧库API改变导致的错误问题。

所以先将Support Library 更新到28，解决了所有API更改到问题，确保项目能在28版本的Support Library下编译通过并通过所有测试。

Support Library28和AndroidX 1.0是等效的二进制文件，意思是两者只是包命名空间的不同：所有的APIs都是一样的。这就意味着从Support Library28迁移到AndroidX，几乎不需要修改什么。

#### Step 2:启用Jetifier

Jetifier用来帮助迁移项目中引用的第三方依赖库使用AndroidX。Jetifier将会改变这些第三方依赖库的字节码，让它们能与使用AndroidX的项目兼容。
在项目中启用`Jetifier`，只需要将下面的代码添加到项目的`gradle.properties`文件中即可：

```bash
android.useAndroidX=true
android.enableJetifier=true
```

然后当同步代码自动导入库的时候，将会导入该库到AndroidX版本，而不是旧到Support Library。

#### Step 3.更新依赖

在开始迁移之前，应该将项目中引用的第三方库更新到该库到最新版本，否则可能会导致编译不通过。

如果使用的是会自动生成代码的第三方库，Jetifier将不会对其进行修改。因此，我们需要检查代码生成库是否与AndroidX兼容。

如果想要直接跳过步骤2和步骤3，则可能会遇到一些错误：

- 如果使用的第三方库的代码与AndroidX不兼容。在这种情况下，如下所示的错误日志将说明它正在尝试获取旧版本的Support Library：



```bash
…
Error : Program type already present: android.support.v4.app.INotificationSideChannel$Stub$Proxy |
Reason: Program type already present: android.support.v4.app.INotificationSideChannel$Stub$Proxy
…
```

- 如果项目只进行了部分的迁移，则可能会遇到重复类的错误，因为它试图从Support Library和AndroidX中提取相同的代码。堆栈跟踪将可能显示以下内容：



```css
…
Duplicate class android.support.v4.app.INotificationSideChannel found in modules classes.jar (androidx.core:core:1.0.0) and classes.jar (com.android.support:support-compat:28.0.0)
…
```

### Step 4:更新项目代码

有两个方案可以来更新项目代码来使用AndroidX：

- Android Studio

- 手动更新

  如果使用的Android Studio版本是3.2以上的，可以通过使用Refactor功能菜单里的Migrate to AndroidX选项去更新代码。这是官方最推荐你的方案，因为Android Studio 可以在重构时会自动检查源码。

  ![img](https://raw.githubusercontent.com/jarrod-chen/jarrod-chen.github.io/master/img/post_20200816_androidx.webp)

  如果不是使用Android Studio开发或者项目结构太复杂导致使用Migrate to AndroidX选项无法涵盖。则可以利用类映射csv文件（在官方文档 [支持库类映射](https://developer.android.google.cn/jetpack/androidx/migrate/class-mappings) 中可查看或下载） ，手动实现查找和替换。将项目中Support Library中的类引用修改成AndroidX中对应的类。

  

  到这里，应该就可以使用AndroidX并能编译通过。


### 意外情况

尽管在这里讨论的工具和流程应该能帮助大多数项目进行顺利的迁移，但是我发现在某些情况下，可能需要手动进行修改。

### 版本配置文件

需要手动重新定义库版本的变量。在迁移之前，build.gradle文件可能是这样的：



```bash
ext.versions = [
    ‘drawer’ : ‘28.0.0’,
    ‘rview’ : ‘28.0.0’
]
implementation “com.android.support:drawerlayout:${versions.drawer}”
implementation “com.android.support:recyclerview-v7:${versions.rview}”
```

在使用迁移工具完成迁移只会，代码可能变成这个样子：



```undefined
ext.versions = [
    ‘drawer’ : ‘28.0.0’,
    ‘rview’ : ‘28.0.0’
]
implementation “androidx.drawerlayout:drawerlayout:1.0.0”
implementation “androidx.recyclerview:recyclerview:1.0.0”
```

DrawerLayout和RecyclerView使用的是AndroidX的包，但是版本号并没有使用到版本号变量，所以需要对其进行手动修改：



```bash
ext.versions = [
    ‘drawer’ : ‘1.0.0’,
    ‘rview’ : ‘1.0.0’
]
implementation “androidx.drawerlayout:drawerlayout:${versions.drawer}”
implementation “androidx.recyclerview:recyclerview:${versions.rview}”
```

### 混淆配置和构建脚本

迁移工具并不会自动更新我们的混淆文件和任何相关的内置脚本。因此，如果有使用到这些文件，并且里面包含Support Library包名称，那就需要手动进行修改。

### 获取最新的稳定版本

迁移工具提取最新版本的AndroidX库。结果可能导致你使用到的某些库是Alpha版本，比如迁移之前，Support Library可能是：



```css
implementation ‘com.android.support:appcompat-v7:28.0.0’
```

迁移之后，可能使用的是Android对应的alpha版本的库：



```css
implementation ‘androidx.appcompat:appcompat:1.1.0-alpha01’
```

所以，最好使用稳定版本的库，那就需要手动去更新以下版本：



```css
implementation ‘androidx.appcompat:appcompat:1.0.2’
```

### 参考

官方文档 “[迁移到AndroidX](https://developer.android.google.cn/jetpack/androidx/migrate)”