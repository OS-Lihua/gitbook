# NeowJava

本部分讨论了使用Java实现智能合约时存在的可能性和限制。尽管我们使用Java进行编程，但生成的字节码并非面向JVM，而是面向NeoVM。因此，只能使用Java的子集，我们称这个子集为NeowJava。

区块链程序员需要谨慎处理计算资源，因为代码中的每一步都会消耗GAS。您应该**避免使用Java的标准库**以及任何未明确为智能合约目的实现的库，因为这些库可能包含不支持的代码或在区块链上执行成本昂贵的代码。

### [类型](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=types)

NeoVM使用称为栈项(stack items)的类型工作 - NeoVM是一个栈机。Neow3j开发包将Java类型映射到这些栈项类型。下表显示了开发包中最重要的Java类型及其对应的NeoVM栈项。

| Java类型                         | 栈项类型       | 描述                                                                                                                |
| ------------------------------ | ---------- | ----------------------------------------------------------------------------------------------------------------- |
| `int`/`Integer`                | Integer    | Java整数在NeoVM上有对应的整数栈项。与普通Java整数的区别在于范围。在NeoVM上，整数范围不限制在\[2-31, 231-1]。我们在下面更深入地讨论整数。                              |
| `boolean`/`Boolean`            | Boolean    | Java的`boolean`映射到NeoVM上的Boolean栈项。但是，NeoVM也使用范围\[0,1]的Integer栈项表示布尔值。如果您的合约方法返回`boolean`但您得到Integer栈项作为返回值，请不要担心。 |
| `byte[]`/`Byte[]`              | Buffer     | Java的字节数组映射到NeoVM上称为Buffer的栈项。这是一个可变字节数组。                                                                         |
| `io.neow3j.devpack.ByteString` | ByteString | 这是一个不可变字节数组。NeoVM的ByteString类型在Java中没有对应类型，因此开发包引入了 `ByteString`。                                                 |
| `java.lang.String`             | ByteString | Java `String`在NeoVM上表示为UTF-8编码的字节数组。栈项类型为ByteString。                                                              |
| `io.neow3j.devpack.Map`        | Map        | NeoVM有专门的映射栈项类型。开发包提供了对应的 `Map` 类型。注意，无法使用`java.utils.Map` 代替。                                                    |
| `io.neow3j.devpack.List`       | Array      | NeoVM有专门的数组栈项类型，用于非字节数组。开发包提供了对应的 `List` 类型。注意，无法使用 `java.utils.List` 代替。                                         |
| 如`int[]`的数组                    | Array      | 除了使用 `List` 外，您也可以使用数组类型，例如 `String[]` 。它们也用Array栈项类型表示。                                                          |
| 自定义对象                          | Array      | 所有其他类，或者说这些类的实例，在NeoVM上表示为Arrays。下面给出更详细的解释。                                                                      |

#### [字节数组](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=byte-arrays)

NeoVM有两个包含字节数组的栈项。ByteString是不可变字节数组，而Buffer是可变的。我们建议默认使用开发包的 `ByteString`，只有在需要可变性时才使用 `byte[]`（它映射到Buffer栈项）。例如，对于方法参数，`ByteString` 是正确的选择，这样可以明确表示方法不会对参数产生任何副作用。您可以使用`toByteArray()` 方法将 `ByteString` 转换为 `byte[]`，反之可以使用 `ByteString(byte[] buffer)` 构造函数。

#### [整数](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=integers)

您可以使用所有Java整数类型，包括它们的包装类。即`byte`、`short`、`char`、`int`、`long` 和 `Byte`、`Short`、`Character`、`Integer`、`Long`。由于NeoVM只知道一种整数栈项，Neow3j编译器将所有这些类型视为相同。它们都有相同的大小限制。将所有整数类型想象为BigIntegers。这意味着Java定义的 `int` 具有32位大小的定义对Neo智能合约不成立。限制要高得多，更准确地说，NeoVM上的整数最多可以占电32字节。我们建议在所有地方使用`int`/`Integer`，忽略 `short`、`char` 和 `long`。`Byte` 和 `byte` 对于表示小值（如常数）很有用。

基于上述，一个问题产生了：“如何初始化大整数？”Java会拒绝您尝试用超出范围\[2-31, 231-1]的值初始化`int` 。为了解决这个问题，您可以使用 `StringLiteralHelper` 及其方法 `stringToInt(String number)`，它允许您初始化任何整数值。

注意，将 `int` 变量转换为 `byte` 时，底层值不会从32位截断到8位。仅仅是表面的Java类型从 `int` 变为`byte` 。如果您使用包装类型方法如 `Integer.byteValue()` 也是一样。开发包提供了两个辅助方法`Helper.asByte(int value)` 和 `Helper.asSignedByte(int value)` ，如果您知道整数值适合字节大小，可以使用它们将 `int` 转换为 `byte`。这些方法不会截断参数，但如果值不适合字节大小会抛出异常。

#### [字符串](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=strings)

您可以使用Java的 `String` 类型来处理文本字符串。但要注意，在NeoVM上 `String` 不是作为对象表示，而是作为UTF8编码的ByteString栈项。这意味着您不能使用 `String` 实例方法，如 `contains()` 或`indexOf()` 。例外是 `length()` 方法，它可以工作，但会给您字符串的字节长度而不是字符数。请使用`io.neow3j.devpack.contracts.StdLib.strLen()` 来获取 `String` 的字符数。

Neow3j支持使用`+`操作符进行字符串连接。但是，不支持混合其他类型。例如，`"hello" + "world"` 可以工作，但 `"hello" + 5` 不行。

查看 `io.neow3j.devpack.Helper` 和 `io.neow3j.devpack.StringLiteralHelper` 类，它们提供了与字符串相关的辅助方法。

#### [数组](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=arrays)

您可以像平常一样使用数组。您只会在多维数组方面面临一些限制。初始化多维数组时，您不能指定除第一个维度以外的所有其他维度的长度。因此，表达式`new String[10][4]`会失败，但`new String[10][]`可以编译。要设置第二个维度，您可以遍历第一个维度并初始化每个位置，如下所示。

```java
String[][] arr = new String[10][];
for (int i = 0; i < arr.length; i++) {
    arr[i] = new String[4];
}Copy to clipboardErrorCopied
```

直接使用特定值进行实例化也是可行的。

```java
String[][] arr = new String[][]{null, new String[]{"hello", " ", "world", "!"}};Copy to clipboardErrorCopied
```

在任何情况下，编译器都会告诉您尝试的操作是否有效。您可以使用 `io.neow3j.devpack.List` 代替Java数组。它比普通数组提供更多便利。它在NeoVM上与数组表示为相同的栈项，因此不会更加昂贵。

#### [对象](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=objects)

上表中未提及的所有其他开发包类的实例，以及您自己定义的类，在NeoVM上都表示为数组。NeoVM数组按照类定义中出现的顺序包含对象的字段变量。即，在以下示例中，`lowNote` 变量在代表该类实例的数组中索引为0，`highNote`的索引为1。

```java
public class Bongo {

    public String lowNote;
    public String highNote;

    public Bongo(String lowNote, String highNote) {
        this.lowNote = lowNote;
        this.highNote = highNote;
    }
}Copy to clipboardErrorCopied
```

开发包本身定义了可以实例化的类。例如，调用 `Storage.getStorageContext()` 返回一个`StorageContext` 实例。在该对象上，您可以调用实例方法，如 `createMap(...)` 。另一个例子是`io.neow3j.devpack.neo.Transaction` 类。使用 `LedgerContract.getTransaction(Hash256 txId)` 检索`Transaction` 对象后，其所有属性通过对象上的公共实例变量对合约类可用。当然，我们可以为类添加getter和setter方法，而不是直接访问成员，但这会在执行合约时产生更高的GAS费用，因为额外的方法调用。

### [合约类](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=contract-class)

NeowJava智能合约由一个主合约类和可能的其他包含主类使用功能的类组成。我们将这个主类称&#x4E3A;_&#x5408;约类_，其他类称&#x4E3A;_&#x8F85;助类_。只有合约类的方法可以访问，并且如果它们是公开的，就会显示在合约清单中。

合约类上的所有内容都是静态的。创建它的实例并将其部署在区块链上的概念在这里不适用。合约的状态不保存在其类变量中，而是通过存储API访问。因此，更应该将合约类视为处理传入调用、合约存储和事件发出的管理实体，而不是实际保持状态本身。这与例如Solidity不同，Solidity中类变量保持合约状态。辅助类也可以是纯静态类，基本上作为合约类使用的函数集合。但是，它们也可以是面向对象意义上的类，在合约类中实例化，为合约数据提供结构和功能。

编译智能合约时，合约类是编译的输入。编译器还会查找并编译合约类中使用的任何辅助类。

#### [合约方法](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=contract-methods)

合约类上的所有方法都需要是静态的。如果您用 `public` 修饰符标记方法，它将显示在合约清单中并可从外部调用。任何其他访问修饰符都会使方法对外部不可见，即，不必使用 `private` 。当然，您可以使用`private` 修饰符使其明确，并禁止任何辅助类调用它。在辅助类中将方法设为公开不会将它们放入合约清单。如果这些类与您的合约类在同一包中，则使用无访问修饰符就足够了。如果它们在另一个包中，则需要像普通Java一样设为公开。

#### [合约变量（静态变量）](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=contract-variables-static-variables)

合约类上的所有变量都必须是静态的。请注意，这些变量不代表合约的状态。它们的值不会写入合约的存储中。NeoVM在合约的每次调用中初始化它们，这意味着您不能使用它们来存储状态。对静态变量的更改会在调用完成后立即丢失。

如果您想禁止更改其值，可以对静态变量应用 `final` 关键字。但是，请注意，用常量值初始化的静态final变量（即，不是从方法调用派生的）将在使用它们的任何地方内联。例如，以下变量不会在合约的其余部分重用，但其值将被复制到使用它的所有脚本位置。

```java
static final int initialSupply = 200_000_000;Copy to clipboardErrorCopied
```

这对小值来说是可以的，但对于在智能合约中经常使用的非常大的值（如长字符串）可能会消耗大量GAS。它会使生成的脚本膨胀，并在调用多次使用该变量的合约方法时产生更高的GAS成本。

在非主合约类的类中，目前仅支持用常量值初始化的final静态变量，如上面的 `initialSupply`。其他静态变量将导致编译器错误。

您可以用常量值（例如，字符串字面量）初始化合约变量，也可以使用方法调用的返回值，如以下示例所示。如上所述，动态初始化的变量在标记为 `final` 时不会内联。

```java
static int initialSupply = 200_000_000;
static String totalSupplyKey = "totalSupply";
static StorageContext sc = Storage.getStorageContext();
static StorageMap assetMap = new StorageMap(sc, "assets");Copy to clipboardErrorCopied
```

Neow3j还支持如下所示的静态初始化子句。但是，实例初始化器，即没有 `static` 关键字的相同子句不受支持。

```java
static final String assetPrefix;
static final StorageContext sc;

static {
    assetPrefix = "asset";
    sc = Storage.getStorageContext();
}Copy to clipboardErrorCopied
```

为了方便变量初始化，开发包提供了 `StringLiteralHelper`。顾名思义，它操作字符串字面量。它提供接受字符串字面量并将其转换为字节数组、脚本哈希或整数的方法。以下示例使用它通过所有者账户的地址设置所有者脚本哈希。

```java
static final Hash160 owner = StringLiteralHelper.addressToScriptHash("NZNos2WqTbu5oCgyfss9kUJgBXJqhuYAaj");Copy to clipboardErrorCopied
```

### [异常](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=exceptions)

Neow3j支持异常和try-catch块，但限制您只能使用 `java.lang.Exception` 类。您可以使用带字符串参数的构造函数（ `new Exception(String message)` ）或不带任何参数的构造函数（ `new Exception()`）。如果没有提供消息，则传递默认消息 `"error"`。

[下面](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=assertions)描述的断言有特殊情况，否则，不允许其他异常类型。此限制意味着您不能有多个catch子句，每个子句处理不同的异常类型。

如果您想在捕获异常时获取异常消息，可以使用方法 `getMessage()`，如以下示例所示：

```java
try {
    throw new Exception("Not allowed.");
} catch (Exception e) {
    e.getMessage(); // equals "Not allowed."
}Copy to clipboardErrorCopied
```

### [断言](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=assertions)

Neow3j支持Java的 `assert` 语句。它在Neo VM上转换为 `ASSERT` 操作码，该操作码不可捕获。如果断言失败，Neo VM会出错，交易会回滚。Neo VM上的操作码不使用消息。因此，不允许向Java的 `assert` 传递消息，在编译时会导致 `CompilerException`。

```java
assert getRemainingBalance() >= 0;Copy to clipboardErrorCopied
```

### [类型比较](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=type-comparison)

如您所知，在Java中，`==` 操作符按值比较原始数据类型，按引用比较复杂类型。在NeowJava中行为不同。使用 `==` 时适用以下规则。

* 按值比较的类型有：`int`、`Integer`、`boolean`、`Boolean`、`String`、`ByteString`、`Hash160`、`Hash256`、`ECPoint`、`Iterator.Struct`。
* 按引用比较的类型有：`byte[]`、`List`、`Map`、数组（例如，`int[]`）、自定义对象。

如果开发包类提供 `equals` 方法，您可以确信它是按值比较。以 `String` 为例。在NeowJava中使用 `==` 比较两个字符串将导致按值比较。使用 `equals` 也是一样的。

### [类型检查](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=type-checking)

neow3j编译器支持以下类型的 `instanceof` 关键字：`int`、`Integer`、`boolean`、`Boolean`、`byte[]`、`String`、`ByteString`、`Hash160`、`Hash256`、`ECPoint`、`Map`、`List`、`Notification`、`InteropInterface`、`Iterator.Struct`、数组。

`instanceof` 操作不支持自定义对象。这是因为NeoVM不会在栈上携带自定义对象的类型信息。它们表示为没有类型信息的数组栈项。如果您对不支持的类型使用 `instanceof`，neow3j编译器将抛出错误。

### [结构体](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=structs)

对于特定类型的集合，您可以创建一个具有不同类型字段变量的类。在NeoVM上，这将引用结构体栈项。为了将类用作结构体，您可以使用注解 `io.neow3j.devpack.annotations.Struct` 。然后您可以设置一个设置其值的构造函数。看看以下示例：

```java
@Struct
public static class ExampleStruct {
    public int id;
    public Hash160 owner;
    public Map<String, Integer> customValues;

    ExampleStruct(int id, Hash160 owner, Map<String, Integer> customValues) {
        this.id = id;
        this.owner = owner;
        this.customValues = customValues;
    }
}Copy to clipboardErrorCopied
```

### [继承](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=inheritance)

每个没有显式扩展另一个类的Java类都是 `Object` 的子类，即使没有重写也可以访问其方法。但是，如果没有显式重写，NeowJava禁止使用 `Object` 方法，如 `toString` 或 `equals` 。

Neow3j支持结构体类的继承，即用 `@Struct` 注解的类。您可以使用关键字 `extends` 从另一个结构体继承字段变量。在以下示例中，通过实例化 `Car` ，在NeoVM上创建的结构体将包含 `brand` 和 `model` 两个变量。

```java
class Vehicle {
    public String brand;

    Vehicle(String brand) {
        this.brand = brand;
    }
}

class Car extends Vehicle {
    public String model;

    Car(String brand, String model) {
        super(brand);
        this.model = model;
    }Copy to clipboardErrorCopied
```

使用 `extends` 关键字的进一步继承支持正在开发中。目前仅适用于结构体和合约接口，如[这里](https://neow3j.io/#/neo-n3/smart_contract_development/devpack?id=smart-contract-interfaces)所述。

### [不支持的功能](https://neow3j.io/#/neo-n3/smart_contract_development/neowjava?id=unsupported-features)

首先，neow3j编译器基于Java 8。因此，不支持在更高版本中添加的任何Java功能。Java 8包含但neow3j编译器不支持的其他功能有：

* 浮点数，即 `float` / `Float` 和 `double` / `Double` 。NeoVM不支持浮点数，因此不能在NeowJava智能合约中使用。一切都用整数处理。
* 枚举
* Lambda表达式
* 接口
