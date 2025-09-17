# 环境搭建与编译

开始Neow3j有两种方式。您可以设置本地环境，也可以使用 GitHub Codespaces 快速开始。如果您想设置本地环境，可以直接跳过GitHub Codespace设置步骤。

使用[样板项目](https://github.com/neow3j/neow3j-boilerplate)仓库来进行基本项目设置，您可以在此基础上构建您的合约。您可以将其用作模板来创建您自己的项目仓库。

<figure><img src="../../../.gitbook/assets/1.gif" alt=""><figcaption></figcaption></figure>

如果您想在本地开发，请跳过下一部分，然后继续[这里](https://neow3j.io/#/neo-n3/smart_contract_development/setup_and_compilation?id=the-build-file)。

### [在GitHub Codespace中设置](https://neow3j.io/#/neo-n3/smart_contract_development/setup_and_compilation?id=setup-in-github-codespace)

您可以通过在GitHub Codespace中使用我们的样板快速开始。这允许您快速开始，而无需设置和管理本地环境。只需遵循以下几个步骤：

在您新创建的样板仓库中启动一个新的codespace。这可能需要一段时间，因为它需要下载和运行多个镜像。

<figure><img src="../../../.gitbook/assets/2.gif" alt=""><figcaption></figcaption></figure>

一旦VSCode IDE在您的浏览器中显示，您应该能看到`main`和`test`源集。`main`包含一个智能合约类，`test`包含一个测试文件。一旦测试类中显示绿色箭头，IDE就完全加载并准备就绪。

> \*\*注意：\*\*还有第三个源集`deploy`，用于生产部署配置。

<figure><img src="../../../.gitbook/assets/3.gif" alt=""><figcaption></figcaption></figure>

现在，您可以通过点击绿色箭头来运行测试。测试类使用neow3j的测试框架，该框架在后台运行一个Neo节点，您的智能合约在此节点上部署和调用。

<figure><img src="../../../.gitbook/assets/4.gif" alt=""><figcaption></figcaption></figure>

### 构建文件

Neow3j开发包使用[Gradle](https://gradle.org/)作为其构建工具。因此，智能合约项目的结构遵循Gradle约定。本部分讨论样板项目中发现的`build.gradle`文件的内容。

文件中的第一个块应用必要的Gradle插件。Neow3j提供了自己的Gradle插件，允许通过名为`neow3jCompile`的Gradle任务进行合约编译。Java插件也是必需的。

```groovy
plugins {
    id 'java'
    id 'io.neow3j.gradle-plugin' version "3.24.0"
}Copy to clipboardErrorCopied
```

然后是项目组织和版本的定义。

```groovy
group 'com.axlabs'
version '1.0-SNAPSHOT'Copy to clipboardErrorCopied
```

接下来，我们必须设置Java版本兼容性，需要是Java 1.8以便Neow3j编译器工作。

```groovy
sourceCompatibility = 1.8
targetCompatibility = 1.8Copy to clipboardErrorCopied
```

然后，我们定义工件仓库来查找代码依赖。默认情况下是Maven Central。Neow3j工件发布在那里。

```groovy
repositories {
    mavenCentral()
}Copy to clipboardErrorCopied
```

接下来建立一个额外的源集。它用于与合约部署相关的代码。这在`main`和`test`之外添加了一个`deploy`源集。

```groovy
sourceSets {
    deploy {
        compileClasspath += sourceSets.main.output
        runtimeClasspath += sourceSets.main.output
    }
}Copy to clipboardErrorCopied
```

然后我们需要定义依赖项，包括用于编写智能合约代码的 `io.neow3j:devpack`，用于编写合约测试的 `io.neow3j:devpack-test` 和 JUnit 5，以及用于在 Java 代码内编译合约的 `io.neow3j:compiler`。

```groovy
dependencies {
    implementation 'io.neow3j:devpack:3.24.0'

    testImplementation 'org.junit.jupiter:junit-jupiter:5.9.0',
            'io.neow3j:devpack-test:3.24.0',
            'ch.qos.logback:logback-classic:1.2.11'

    deployImplementation 'io.neow3j:compiler:3.24.0',
            'ch.qos.logback:logback-classic:1.2.11'
}Copy to clipboardErrorCopied
```

为了让 JUnit 5 与 Gradle 协同工作，我们需要添加以下代码块。

```groovy
tasks.withType(Test) {
    useJUnitPlatform()
}Copy to clipboardErrorCopied
```

最后，有一个名为 `neow3jCompiler` 的代码块，它配置了 `neow3jCompile` Gradle 任务。我们要编译的合约通过 `className` 属性以其完全限定名声明。这里有两个未使用的隐藏属性。第一个是 `debug` 属性，它决定编译是否应该为 Neo 调试器生成调试信息。默认值为 true。第二个是 `outputDir`，它指定放置编译结果的目录。默认值为 `build/neow3j`。

```groovy
neow3jCompiler {
    className = "com.axlabs.boilerplate.HelloWorldSmartContract"
}Copy to clipboardErrorCopied
```

### [编译](https://neow3j.io/#/neo-n3/smart_contract_development/setup_and_compilation?id=compilation)

合约编译可以通过两种方式进行。您可以使用 neow3j 的 Gradle 任务，或者从 Java 代码内调用编译器。接下来的两个部分将解释这两种方法。

#### [使用 Gradle](https://neow3j.io/#/neo-n3/smart_contract_development/setup_and_compilation?id=using-gradle)

通过上述设置，我们现在可以开始编译。样板项目包含一个非常简单的合约。要编译它，请从项目根文件夹运行 `./gradlew neow3jCompile`。这将编译智能合约类，并将 NEF 文件、合约清单和调试信息文件输出到输出目录 `build/neow3j`。NEF 文件是包含 NeoVM 代码的合约二进制文件。合约清单定义了合约的属性，例如，哪些方法可用于调用。以 `.nefdbgnfo` 结尾的调试信息文件是 Neo 调试器调试您的合约所需的。NEF 和清单是我们部署所需的产物。

#### [程序化编译](https://neow3j.io/#/neo-n3/smart_contract_development/setup_and_compilation?id=programmatic-compilation)

neow3j 编译器可以从另一个 Java 程序内调用。为此，我们需要 `io.neow3j:compiler` 模块，该模块已包含在样板项目中。

在类 `com.axlabs.boilerplate.Deployment` 中，您可以看到 neow3j 编译器的示例用法。

```java
CompilationUnit res = new Compiler().compile(
               HelloWorldSmartContract.class.getCanonicalName(),
               substitutions);Copy to clipboardErrorCopied
```

编译需要要编译的合约类的完全限定名，并将在项目中查找该类文件。编译后，结果可在 `CompilationUnit` 中访问。它包含 NEF 文件、合约清单和其他信息。

```java
NefFile nef = result.getNefFile();
ContractManifest manifest = result.getManifest();Copy to clipboardErrorCopied
```

如果您希望编译器生成调试信息，您需要告诉 neow3j 编译器智能合约的源代码位置。合约的 Java 文件是位于同一项目中还是其他地方都无关紧要。

```java
File sourceDirectory = new File("/path/to/the/contract/src/java/main");
DirectorySourceContainer sourceContainer = new DirectorySourceContainer(sourceDirectory, false);
CompilationUnit result = new Compiler()
        .compile("com.axlabs.boilerplate.HelloWorldSmartContract", Arrays.asList(sourceContainer));
DebugInfo debugInfo = result.getDebugInfo();
```

