[//]: # (title: Add dependencies to KMM modules)
[//]: # (auxiliary-id: Add_dependencies_to_KMM_modules)

每个应用程序都需要一些库才能良好运转。
Kotlin 移动多平台（KMM）应用程序既可以依赖于能同时在 iOS 和 Android 上运行的多平台库，也可以依赖于 iOS 和 Android 各自平台特有的库。

在这里你可以学到如何添加：
* [多平台依赖项](#多平台库)
* [iOS 依赖项](#ios-依赖项))
* [Android 依赖项](#android-依赖项)

## 多平台库

你可以添加采用了 Kotlin 多平台技术的库的依赖项，例如<!--
--> [kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines) 与 [SQLDelight](https://github.com/cashapp/sqldelight)。
这些库的作者通常会提供将其依赖项添加到项目的指南。

> 当在一个有[分层项目结构支持](https://kotlinlang.org/docs/reference/mpp-share-on-platforms.html#share-code-on-similar-platforms)的多平台项目里使用了一个没有分层项目结构支持的多平台库时，
> 你将不能在共享的 iOS source set 中使用 IDE 的特性，例如代码补全和高亮提示。
> 
> 这是一个[已知问题](https://youtrack.jetbrains.com/issue/KT-40975)，我们正在努力解决它。与此同时，你可以使用[这个变通方案](#为共享的-ios-source-set-启用-ide-支持的变通方案)。
>
{type="note"}

本页涵盖了基本的依赖项使用示例：

* [针对 Kotlin 标准库](#针对-kotlin-标准库的依赖项)
* [针对所有 source set 共享的库](#针对所有-source-set-共享的库的依赖项)
* [针对指定 source set 中使用的库](#针对指定-source-set-中使用的库的依赖项)
* [针对另一个多平台项目](#针对另一个多平台项目的依赖项)

了解更多关于[配置依赖项](https://kotlinlang.org/docs/reference/using-gradle.html#configuring-dependencies)的信息。

查看[社区维护的 Kotlin 多平台库列表](https://libs.kmp.icerock.dev/)。

### 针对 Kotlin 标准库的依赖项

Kotlin 标准库会自动添加到所有多平台项目中，无需手动执行任何操作。

### 针对所有 source set 共享的库的依赖项

如果想从任一个 source set 中使用库，那么可以只将其添加到公共 source set 中。
Kotlin 移动多平台插件将会自动地把相应的部分添加到任何其他 source set 中。

<tabs>
<tab title="Groovy">
    
```Groovy
kotlin {
    sourceSets {
        commonMain {
            dependencies {
                implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:%coroutinesVersion%'
            }
        }
        androidMain {
            dependencies {
                //kotlinx.coroutines 平台部分的依赖项将被自动添加
            }
        }
    }
}
```
        
</tab>
<tab title="Kotlin">
    
```Kotlin
kotlin {
    sourceSets["commonMain"].dependencies {
        implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:%coroutinesVersion%")
    }
    sourceSets["androidMain"].dependencies {
        //kotlinx.coroutines 平台部分的依赖项将被自动添加
    }
}
```
</tab>
</tabs>

### 针对指定 source set 中使用的库的依赖项

如果想让一个多平台库仅用于某个指定的 source set，那么可以将其专门添加到其中。
之后这个指定的库将只能在那些 source set 中使用。
   
> 这种情况下不要使用特定于平台的名称，如下面示例中的 SQLDelight `native-drive`。在库的文档中找到确切的名称。
> 
{type="note"}   

<tabs>
<tab title="Groovy">
    
```Groovy
kotlin {
    sourceSets {
        commonMain {
            dependencies { 
            // kotlinx.coroutines 将在所有 source set 中可用
            implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:%coroutinesVersion%'
            }
        }
        androidMain {
            dependencies { }
        }
        iosMain {
            dependencies {
            // SQLDelight 将仅在 iOS source set 中可用，而在 Android 或公共 source set 中不可用   
            implementation 'com.squareup.sqldelight:native-driver:%sqlDelightVersion%'
            }
        }
    }
}
```
        
</tab>
<tab title="Kotlin">
    
```Kotlin
kotlin {
    sourceSets["commonMain"].dependencies {
        //kotlinx.coroutines 将在所有 source set 中可用
        implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:%coroutinesVersion%")
    }
    sourceSets["androidMain"].dependencies {
    }
    sourceSets["iosX64Main"].dependencies {
        //SQLDelight 将仅在 iOS source set 中可用，而在 Android 或公共 source set 中不可用
        implementation("com.squareup.sqldelight:native-driver:%sqlDelightVersion%")
    }
}
```
</tab>
</tabs>

### 针对另一个多平台项目的依赖项

可以将一个多平台项目作为依赖项连接到另一个多平台项目。为此，只需将项目依赖项添加到需要它的 source set 中。
如果想要在所有 source set 中使用这个依赖项，就将其添加到公共 source set 中。在这种情况下，其他 source set 将自动获得其版本。

<tabs>
<tab title="Groovy">
    
```Groovy
kotlin {
    sourceSets {
        commonMain {
            dependencies {
                implementation project(':some-other-multiplatform-module')
            }
        }
        androidMain {
            dependencies {
                //:some-other-multiplatform-module 的平台部分将会被自动添加
            }
        }
    }
}
```
        
</tab>
<tab title="Kotlin">
    
```Kotlin
kotlin {
    sourceSets["commonMain"].dependencies {
        implementation(project(":some-other-multiplatform-module"))
    }
    sourceSets["androidMain"].dependencies {
        //:some-other-multiplatform-module 的平台部分将会被自动添加
    }
}
```
</tab>
</tabs>

## iOS 依赖项

Apple SDK 依赖项（例如 Foundation 或 Core Bluetooth）作为 Kotlin Multiplatform Mobile 项目中的一组预构建库提供。
它们不需要任何额外的配置。

也可以在 iOS source set 中复用 iOS 生态系统中的其他库和框架。
Kotlin 提供了与 Objective-C 依赖项的交互能力，Swift 依赖项也可以在 Kotlin 中使用如果它的 APIs 用 @objc 导出为 Objective-C。
纯 Swift 依赖项目前还不支持。

CocoaPods 依赖项管理器的集成也受到同样的限制——不能使用纯 Swift pod。

我们推荐在 Kotlin 移动多平台（KMM）项目中[使用 CocoaPods](#使用-CocoaPods)去处理 iOS 依赖项。
仅当你明确地想调整（两门语言）互操作的过程或有其他强有力的理由这样做时,才去[手动管理依赖关系](#不使用-CocoaPods)。

> 当在一个有[分层项目结构支持](https://kotlinlang.org/docs/reference/mpp-share-on-platforms.html#share-code-on-similar-platforms)的多平台项目（例如使用了 `ios()` [目标平台快捷方式](https://kotlinlang.org/docs/reference/mpp-share-on-platforms.html#use-target-shortcuts)）中使用 iOS 第三方库时，
> 将不能在共享的 iOS source set 中使用 IDE 的特性，例如代码补全和高亮提示。
> 
> 这是一个[已知问题](https://youtrack.jetbrains.com/issue/KT-40975)，我们正在努力解决它。同时，你可以尝试[这个变通方案](#为共享的-ios-source-set-启用-ide-支持的变通方案)。
>
> 这个问题并不适用于某些开箱即用的[平台库](https://kotlinlang.org/docs/native-platform-libs.html)。
>
{type="note"}

### 使用 CocoaPods

1. 执行 [初始化 CocoaPods 集成设置](https://kotlinlang.org/docs/reference/native/cocoapods.html#install-the-cocoapods-dependency-manager-and-plugin)

2. 通过编写项目构建脚本中的 `pod()`，添加要使用的 CocoaPods 版本库中的 Pod 库依赖项。

    <tabs>
    <tab title="Groovy">

    ```Groovy
    kotlin {
        cocoapods {
            //..
            pod('AFNetworking') {
                version = '~> 4.0.1'
            }
        }
    }
    ```

    </tab>
    <tab title="Kotlin">

    ```Kotlin
    kotlin {
        cocoapods {
            //..
            pod("AFNetworking") {
                version = "~> 4.0.1"
            }
        }
    }
    ```

    </tab>
    </tabs>

3. 重新导入项目。

如需在 Kotlin 代码中使用该依赖项，可导入软件包 `cocoapods.<library-name>`。在上面的示例中，即：
```kotlin
import cocoapods.AFNetworking.*
```

了解更多关于 [CocoaPods 集成](https://kotlinlang.org/docs/reference/native/cocoapods.html)。

### 不使用 CocoaPods

如果不想使用 CocoaPods，那么可以使用 cinterop 工具为 Objective 或 Swift 的声明创建 Kotlin 绑定。这样就可以通过 Kotlin 代码去调用它们。为此：
1. 下载依赖项。
2. 构建并获取它的二进制文件。
3. 创建一个具体的 `.def` 文件来描述 cinterop 的依赖项。
4. 调整构建脚本以在构建期间生成绑定。

[代码库](#不使用-cocoapods-添加库)和 [frameworks](#不使用-cocoapods-添加一个-framework) 的步骤有所不同，但是它们的思想是一样的。

#### 不使用 CocoaPods 添加库

1. 下载库源代码，并将其放在可以从项目中引用到它的地方。

2. 构建这个库（库作者通常会提供构建指南），并取得二进制文件的路径。

3. 在项目里创建一个 `.def` 文件，例如 `DateTools.def`。

4. 在该文件的第一行添加：`language = Objective-C`。如果要使用纯 C 依赖项，就省略这个语言属性。

5. 为下面两个必要的属性赋值：
    * `headers` 描述哪些头文件将被 cinterop 处理。
    * `package` 设置这些声明应该被放入的包的名称。

    例如:
    ```properties
    headers = DateTools.h
    package = DateTools
    ```

6. 在构建脚本中添加关于这个库互操作性的信息：
    * 添加 `.def` 文件的路径。如果 `.def` 文件与 cinterop 具有相同的名称，并且位于 `src/nativeInterop/cinterop/` 目录中，那么可以忽略此路径。
    * 使用 `includeDirs` 选项告诉 cinterop 在哪里查找头文件。
    * 配置库的二进制文件的链接。

    <tabs>
    <tab title="Groovy">

    ```Groovy
    kotlin {
        iosX64 {
            compilations.main {
                cinterops {
                    DateTools {
                        // .def 文件的路径
                        defFile("src/nativeInterop/cinterop/DateTools.def")
                   
                        // 头文件检索的目录（类似于 -I<path> 的编译器选项）
                        includeDirs("include/this/directory", "path/to/another/directory")
                    }
                    anotherInterop { /* ... */ }
                }
            }

            binaries.all {
                // 链接到库所需的链接器选项。
                linkerOpts "-L/path/to/library/binaries", "-lbinaryname"
            }
        }
    }
    ```

    </tab>
    <tab title="Kotlin">

    ```Kotlin
    kotlin {
        iosX64() {
            compilations.getByName("main") {
                val DateTools by cinterops.creating {
                    // .def 文件的路径
                    defFile("src/nativeInterop/cinterop/DateTools.def")

                    // 头文件检索的目录（类似于 -I<path> 的编译器选项）
                    includeDirs("include/this/directory", "path/to/another/directory")
                }
                val anotherInterop by cinterops.creating { /* ... */ }
            }

            binaries.all {
                // 链接到库所需的链接器选项。
                linkerOpts("-L/path/to/library/binaries", "-lbinaryname")
            }
        }
    }
    ```

    </tab>
    </tabs>

7. 构建项目.

现在你可以在你的 Kotlin 代码中使用该依赖项，即导入那个之前在 .def 文件里的 package 配置对应的包。在上面的示例中，这将是：
```kotlin
import DateTools.*
```

#### 不使用 CocoaPods 添加一个 framework

1. 下载 framework 源代码，并将其放在可以从项目中引用到它的地方。

2. 构建这个 framework（framework 作者通常会提供构建指南），并取得二进制文件的路径。

3. 在项目里创建一个 `.def` 文件，例如 `MyFramework.def`。

4. 在该文件的第一行添加：`language = Objective-C`。如果要使用纯 C 依赖项，就省略这个语言属性。

5. 为下面两个必要的属性赋值：
    * `modules` – 需要被 cinterop 处理的 framework 的名称.
    * `package` – 这些声明将要被放入的包的名称.
    示例:
    ```properties
    modules = MyFramework
    package = MyFramework
    ```

6.  在构建脚本中添加关于这个 framework 互操作性的信息:
    * 添加 `.def` 文件的路径。如果 `.def` 文件与 cinterop 具有相同的名称，并且位于 `src/nativeInterop/cinterop/` 目录中，那么可以忽略此路径。
    * 使用 `-framework` 选项将 framework 的名称传递给编译器和链接器。
    使用 `-F` 选项将 framework 源和二进制文件的路径传递给编译器和链接器。

    <tabs>
    <tab title="Groovy">
    
    ```Groovy
    kotlin {
        iosX64 {
            compilations.main {
                cinterops {
                    DateTools {
                        // .def 文件的路径
                        defFile("src/nativeInterop/cinterop/MyFramework.def")
                   
                        compilerOpts("-framework", "MyFramework", "-F/path/to/framework/")
                    }
                    anotherInterop { /* ... */ }
                }
            }

            binaries.all {
                // 告诉链接器 framework 的位置。
                linkerOpts("-framework", "MyFramework", "-F/path/to/framework/")
            }
        }
    }

    ```
    </tab>
    <tab title="Kotlin">
    
    ```Kotlin
    kotlin {
        iosX64() {
            compilations.getByName("main") {
                val DateTools by cinterops.creating {
                    // .def 文件的路径
                    defFile("src/nativeInterop/cinterop/DateTools.def")

                   compilerOpts("-framework", "MyFramework", "-F/path/to/framework/"
               }
               val anotherInterop by cinterops.creating { /* ... */ }
            }

            binaries.all {
                // 告诉链接器 framework 的位置。
                linkerOpts("-framework", "MyFramework", "-F/path/to/framework/")
            }
       }
    }
    ```

    </tab>
    </tabs>

7. 构建项目。

现在你可以在你的 Kotlin 代码中使用该依赖项，即导入那个之前在 .def 文件里的 package 配置对应的包。在上面的示例中，这将是：

```kotlin
import MyFramework.*
```

了解更多关于 [Objective-C 和 Swift 互操作性](https://kotlinlang.org/docs/reference/native/objc_interop.html)和<!--
-->[用 Gradle 配置 cinterop](https://kotlinlang.org/docs/reference/mpp-dsl-reference.html#cinterops)的信息。

### 为共享的 iOS source set 启用 IDE 支持的变通方案 {initial-collapse-state="collapsed"}

由于一个[已知问题](https://youtrack.jetbrains.com/issue/KT-40975)，如果你的多平台项目<!--
-->使用了[分层项目结构支持](https://kotlinlang.org/docs/reference/mpp-share-on-platforms.html#share-code-on-similar-platforms)并且有如下所示的依赖项，你将无法在共享的 iOS source set 中使用 IDE 特性，例如代码补全和高亮显示：

* 不支持分层项目结构的多平台库
* 除了开箱即用的[平台库](https://kotlinlang.org/docs/native-platform-libs.html)之外的第三方 iOS 库。

该问题仅限于共享的 iOS source set。IDE 将正确支持其余代码。

> 所有使用 Kotlin 移动多平台（KMM）项目向导创建的项目都支持分层项目结构，所以它们会受到该问题的影响。
>
{type="note"}

如需在这些情况下开启 IDE 支持，可通过在项目的 `shared` 目录中的 `build.gradle.(kts)` 文件里添加如下代码解决该问题：

<tabs>

```Groovy
def iosTarget
if (System.getenv("SDK_NAME")?.startsWith("iphoneos")) {
    iosTarget = kotlin.&iosArm64
} else {
    iosTarget = kotlin.&iosX64
}
```

```Kotlin
val iosTarget: (String, KotlinNativeTarget.() -> Unit) -> KotlinNativeTarget =
    if (System.getenv("SDK_NAME")?.startsWith("iphoneos") == true)
        ::iosArm64
    else
        ::iosX64

iosTarget("ios")
```

</tabs>

在此代码示例中，iOS 目标平台的配置取决于 `SDK_NAME` 环境变量，该变量由 Xcode 管理。
在每次构建中，你将只有一个使用了 `iosMain` source set 的 iOS 目标，叫做 `ios`。
这里将没有 `iosMain`，`iosArm64`，`iosX64` source set 的分层。

> 这是一个临时的变通方案。如果你是一个软件库作者，我们推荐尽快[迁移到分层项目结构](https://kotlinlang.org/docs/migrating-multiplatform-project-to-14.html#migrate-to-the-hierarchical-project-structure)。
>
> 通过这种变通方案，Kotlin 多平台工具仅针对当前构建期间处于活动状态的一个原生目标平台来分析您的代码。
> 这可能会在所有目标平台完成构建期间导致各种错误，并且如果项目除 iOS 之外还包含其他原生目标平台，那么将更可能发生错误。
>
{type="note"}

## Android 依赖项

将 Android 平台上的依赖项添加到 Kotlin 移动多平台（KMM）模块的工作流程与纯 Android 项目相同：在 Gradle 构建脚本中添加一行代码来声明所需的依赖项并导入项目中。然后就可以在 Kotlin 代码中使用此依赖项。

我们建议通过将 Android 依赖项添加到具体的 Android source set 中来将其集成到 Kotlin 移动多平台（KMM）项目中。

<tabs>
<tab title="Groovy">
    
```Groovy
sourceSets {
    androidMain {
        dependencies {
            implementation 'com.example.android:app-magic:12.3'
        }
    }
}
```

</tab>
<tab title="Kotlin">

```Kotlin
sourceSets["androidMain"].dependencies {
        implementation("com.example.android:app-magic:12.3")
}
```

</tab>
</tabs>

如果一个 Android 项目中的顶层依赖项有一个特殊的配置名称，那么将这个项目中的一个顶层依赖项移动到 Kotlin 移动多平台（KMM）项目中的指定 source set 里可能会很困难。例如，要在 Android 项目的顶层移动一个 `debugImplementation` 依赖项，需要向名为 `androidDebug` 的 source set 添加实现依赖项。
为了最大程度地减少处理此类迁移问题的工作量，可以在 `android` 块内添加一个  `dependencies` 块：

<tabs>
<tab title="Groovy">

```Groovy
android {
    ...

    dependencies {
        implementation 'com.example.android:app-magic:12.3'
    }
}
```
        
</tab>
<tab title="Kotlin">
    
```Kotlin
android {
    ...

    dependencies {   
        implementation("com.example.android:app-magic:12.3")
    }
}
```
</tab>
</tabs>

这里声明的依赖项将与顶级块中的依赖项完全相同，但是以这种方式声明可以在构建脚本中直观地分离 Android 依赖项，从而减少混乱。

还支持以 Android 项目惯用的方式，将依赖项放入脚本末尾的独立 `dependencies` 块中。但是，我们强烈**反对**这样做，因为在构建脚本的顶层代码块中配置一个 Android 依赖项而在每个 source set 中配置其他目标平台依赖项，将很可能会引发混淆。

前往 Android 官方文档了解更多关于[添加依赖项](https://developer.android.com/studio/build/dependencies)的信息。
