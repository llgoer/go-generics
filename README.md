# Go：为何带来泛型

- 中文版 

- [English version](README-en.md)

## 介绍

[这是在Gophercon 2019上发表的演讲版本。视频链接可供使用。]

这篇文章是关于向Go添加泛型的意义，以及为什么我认为我们应该这样做。我还将介绍为Go添加泛型的设计可能的改变。

Go于2009年11月10日发布。不到24小时后，我们看到了[关于泛型](https://groups.google.com/d/msg/golang-nuts/70-pdwUUrbI/onMsQspcljcJ)的[第一条评论](https://groups.google.com/d/msg/golang-nuts/70-pdwUUrbI/onMsQspcljcJ)。（该评论还提到了我们在2010年初以panic和recover的形式添加到语言中的情况。）

在Go调查的三年中，缺乏泛型一直被列为该语言需要修复的三大问题之一。

------

本项目会带来更多关于Go泛型的文档、代码及讨论。

## 为什么是泛型？

但是添加泛型是什么意思，为什么我们想要呢？

用 [Jazayeri等人的话来说](https://www.dagstuhl.de/en/program/calendar/semhp/?semnr=98171)：泛型编程能够以通用类型的形式表示函数和数据结构。

那意味着什么呢？

举一个简单的例子，我们假设我们想要反转切片中的元素。这不是很多程序需要做的事情，但并不是那么不寻常。

让我们说它是一个int slice。

```go
func ReverseInts(s []int) {
    first := 0
    last := len(s)
    for first < last {
        s[first], s[last] = s[last], s[first]
        first++
        last--
    }
}
```

非常简单，但即使是一个简单的功能，你也想写一些测试用例。事实上，当我这样做时，我发现了一个错误。我相信很多读者已经发现了它。

```go
func ReverseInts(s []int) {
    first := 0
    last := len(s) - 1
    for first < last {
        s[first], s[last] = s[last], s[first]
        first++
        last--
    }
}
```

我们在最后设置变量时需要减去1。

现在让我们反转一段字符串。

```go
func ReverseStrings(s []string) {
    first := 0
    last := len(s) - 1
    for first < last {
        s[first], s[last] = s[last], s[first]
        first++
        last--
    }
}
```
如果比较ReverseInts以及ReverseStrings，您将看到两个函数完全相同，除了参数的类型。我认为没有任何读者对此感到惊讶。

有些人对Go感到惊讶的是，没有办法编写Reverse适用于任何类型切片的简单函数。

大多数其他语言都可以让你编写这种功能。

在像Python或JavaScript这样的动态类型语言中，您可以简单地编写函数，而无需指定元素类型。这在Go中不起作用，因为Go是静态类型的，并且要求您记下切片的确切类型和切片元素的类型。

大多数其他静态类型语言，如C++或Java或Rust或Swift，支持泛型来解决这类问题。

## 今天的Go通用编程

那么人们如何在Go中编写这种代码呢？

在Go中，您可以使用接口类型编写适用于不同切片类型的单个函数，并在要传递的切片类型上定义方法。这就是标准库的sort.Sort功能的工作方式。

换句话说，Go中的接口类型是通用编程的一种形式。他们让我们捕捉不同类型的共同方面，并将它们表达为方法。然后我们可以编写使用这些接口类型的函数，这些函数将适用于实现这些方法的任何类型。

但这种做法不符合我们的要求。使用接口，您必须自己编写方法。使用几种方法定义命名类型来反转切片是很尴尬的。你编写的方法对于每种切片类型都是完全相同的，所以从某种意义上说，我们只是移动并压缩了重复的代码，我们还没有消除它。虽然接口是泛型的一种形式，但它们并没有向我们提供我们想要的所有泛型。

使用泛型接口的另一种方法是，可以解决自己编写方法的需要，就是让语言为某些类型定义方法。这不是语言支持的东西，但是，例如，语言可以定义每个切片类型都有一个返回元素的Index方法。但是为了在实践中使用该方法，它必须返回一个空接口类型，然后我们失去了静态类型的所有好处。更巧妙的是，没有办法定义一个泛型函数，该函数采用具有相同元素类型的两个不同切片，或者采用一个元素类型的映射并返回相同元素类型的切片。Go是一种静态类型语言，因为这样可以更容易地编写大型程序; 我们不想失去静态类型的好处，以获得泛型的好处。

另一种方法是Reverse使用反射包编写一个泛型函数，但这样写起来很笨拙而且运行速度慢，很少有人这样做。该方法还需要显式类型断言，并且没有静态类型检查。

或者，您可以编写一个代码生成器，它接受一个类型并Reverse为该类型的切片生成一个函数。有几个代码生成器就是这样做的。但是这为需要的每个包添加了另一个步骤Reverse，它使构建变得复杂，因为必须编译所有不同的副本，并且修复主源中的错误需要重新生成所有实例，其中一些实例可能完全在不同的项目中。

所有这些方法都很尴尬，我认为大多数必须在Go中反转切片的人只是为他们需要的特定切片类型编写函数。然后他们需要为函数编写测试用例，以确保它们没有像我最初制作的那样犯一个简单的错误。而且他们需要定期运行这些测试。

但是我们这样做，这意味着除了元素类型之外，对于看起来完全相同的函数来说，还有很多额外的工作。并不是说它无法完成。显然可以做到，Go程序员正在这样做。只是应该有更好的方式。

对于像Go这样的静态类型语言，更好的方法是泛型。我之前写的是泛型编程能够以通用形式表示函数和数据结构，并将类型考虑在内。这正是我们想要的。

## 为Go可以带来什么样的泛型

我们想要从Go中的泛型中获得的第一个也是最重要的事情是能够编写函数，Reverse而不必关心切片的元素类型。我们想要分解出那种元素类型。然后我们可以编写一次函数，编写测试一次，将它们放在一个go-gettable包中，并随时调用它们。

更好的是，由于这是一个开源世界，其他人可以写Reverse一次，我们可以使用他们的实现。

在这一点上，我应该说“泛型”可能意味着很多不同的东西。在本文中，我所说的“泛型”是我刚才所描述的。特别是，我不是指C ++语言中的模板，它支持的内容比我在这里写的要多得多。

我Reverse详细介绍了，但是我们可以编写许多其他功能，例如：

- 查找切片中的最小/最大元素
- 求出切片的平均值/标准差
- 计算联合/交叉的maps
- 在node/edge中查找最短路径
- 将转换函数应用于slice/map，返回新的slice/map

这些示例以大多数其他语言提供。实际上，我通过浏览C++标准模板库来编写这个列表。

还有一些特定于Go的示例，它强烈支持并发性。

- 从具有超时的通道读取
- 将两个通道组合成一个通道
- 并行调用函数列表，返回一片结果
- 调用函数列表，使用Context，返回第一个函数的结果来完成，取消和清理额外的goroutines

我已经看到所有这些函数用不同的类型写了很多次。在Go中编写它们并不难。但是能够重用适用于任何值类型的高效且便于调试的实现会很好。

需要说明的是，这些仅仅是一些例子。还有许多通用功能可以使用泛型更容易和安全地编写。

另外，正如我之前所写，它不仅仅是功能。它也是数据结构。

Go有两种内置于该语言中的通用通用数据结构：切片和maps。切片和maps可以保存任何数据类型的值，使用静态类型检查存储和检索的值。值存储为自身，而不是接口类型。也就是说，当我有a时[]int，切片直接保存int，而不是int转换为接口类型。

切片和maps是最有用的通用数据结构，但它们不是唯一的。以下是其他一些例子。

- Sets
- 自平衡树，有序插入和按行排序遍历
- Multimaps，具有密钥的多个实例
- 并发哈希映射，支持并行插入和查找，没有单个锁

如果我们可以编写泛型类型，我们可以定义像这样的新数据结构，它们具有与切片和映射相同的类型检查优势：编译器可以静态地类型检查它们所持有的值的类型，并且值可以存储为自身，而不是存储为接口类型。

还应该可以采用前面提到的算法并将它们应用于通用数据结构。

这些示例应该就像Reverse：通用函数和数据结构一次编写，在包中，并在需要时重用。它们应该像切片和maps一样工作，因为它们不应该存储空接口类型的值，但是应该存储特定的类型，并且应该在编译时检查这些类型。

这就是Go可以从泛型中获得的东西。泛型可以为我们提供强大的构建块，让我们可以更轻松地共享代码和构建程序。

我希望我已经解释了为什么值得研究。

## 好处和成本

但泛型并非来自[Big Rock Candy Mountain](https://mainlynorfolk.info/folk/songs/bigrockcandymountain.html)，那里每天阳光照射在[柠檬水泉上](http://www.lat-long.com/Latitude-Longitude-773297-Montana-Lemonade_Springs.html)。每一种语言变化都有成本。毫无疑问，向Go添加泛型将使语言更复杂。与语言的任何变化一样，我们需要谈论最大化利益并最大限度地降低成本。

在Go中，我们的目标是通过可以自由组合的独立，正交语言功能来降低复杂性。我们通过简化各个功能来降低复杂性，并通过允许其自由组合来最大化功能的好处。我们希望对泛型做同样的事情。

为了使这个更具体，我将列出一些我们应该遵循的准则。

*尽量减少新概念

我们应该尽可能少地为语言添加新概念。这意味着最少的添加新语法和最少的新关键字和其他名称。

*复杂性落在通用代码的编写者身上，而不是用户身上

编程通用包的程序员应尽可能地降低复杂性。我们不希望包的用户不必担心泛型。这意味着应该可以以自然的方式调用泛型函数，同时使用通用包时的任何错误都应该以易于理解和修复的方式报告。将调用调用为通用代码也应该很容易。

*作家和用户可以独立工作

同样，我们应该很容易将通用代码的编写者及其用户的关注点分开，以便他们可以独立开发代码。他们不应该担心对方在做什么，不仅仅是不同包中正常功能的编写者和调用者都要担心。这听起来很明显，但对于其他所有编程语言中的泛型都不是这样。

*构建时间短，执行时间短

当然，尽可能地，我们希望保持Go给我们今天的短建造时间和快速执行时间。泛型倾向于在快速构建和快速执行之间进行权衡。我们尽可能地想要两者。

*保持Go的清晰度和简洁性

最重要的是，Go今天是一种简单的语言。Go程序通常清晰易懂。我们探索这个空间的漫长过程的一个主要部分是试图了解如何在保持清晰度和简洁性的同时添加泛型。我们需要找到适合现有语言的机制，而不是把它变成完全不同的东西。

这些指南应适用于Go中的任何泛型实现。这是我今天想要留给你的最重要的信息： 泛型可以为语言带来显着的好处，但是如果Go仍然感觉像Go那么它们是值得做的。

## 草案设计

幸运的是，我认为可以做到。为了完成这篇文章，我将讨论为什么我们想要泛型，以及对它们的要求是什么，简要讨论我们认为如何将它们添加到语言中的设计。

在今年的Gophercon Robert Griesemer和我发布了[一个设计草案](https://github.com/golang/proposal/blob/master/design/go2draft-contracts.md)，为Go添加泛型。有关详细信息，请参阅草案。我将在这里讨论一些要点。

这是此设计中的通用反向函数。

```go
func Reverse (type Element) (s []Element) {
    first := 0
    last := len(s) - 1
    for first < last {
        s[first], s[last] = s[last], s[first]
        first++
        last--
    }
}
```
您会注意到函数的主体完全相同。只有签名发生了变化。

已经考虑了切片的元素类型。它现在被命名Element并成为我们所谓的 类型参数。它不是成为slice参数类型的一部分，而是一个单独的附加类型参数。

要使用类型参数调用函数，在一般情况下，您传递一个类型参数，这与任何其他参数类似，只不过它是一个类型。

```go
func ReverseAndPrint(s []int) {
    Reverse(int)(s)
    fmt.Println(s)
}
```

这是在这个例子中就是你看到的Reverse后面的(int)。

幸运的是，在大多数情况下，包括这个，编译器可以从常规参数的类型中推断出类型参数，并且根本不需要提及类型参数。

调用泛型函数就像调用任何其他函数一样。

```go
func ReverseAndPrint(s []int) {
    Reverse(s)
    fmt.Println(s)
}
```
换句话说，虽然通用Reverse功能略高于更加复杂ReverseInts和ReverseStrings，这种复杂落在函数上，而不是编写和调用。

## 合约

由于Go是一种静态类型语言，我们必须讨论类型参数的类型。这个元类型告诉编译器在调用泛型函数时允许哪种类型的参数，以及泛型函数可以对类型参数的值执行哪些操作。

该Reverse函数可以使用任何类型的切片。它对类型值的唯一作用Element是赋值，它适用于Go中的任何类型。对于这种通用函数，这是一种非常常见的情况，我们不需要对类型参数说些什么特别的。

让我们快速浏览一下不同的功能。

```go
func IndexByte (type T Sequence) (s T, b byte) int {
    for i := 0; i < len(s); i++ {
        if s[i] == b {
            return i
        }
    }
    return -1
}
```

目前，标准库中的bytes包和strings包都有一个IndexByte函数。此函数返回b序列中的索引s，其中s是一个string或一个[]byte。我们可以使用这个单一的泛型函数来替换字节和字符串包中的两个函数。在实践中，我们可能不会这样做，但这是一个有用的简单示例。

在这里，我们需要知道类型参数的T作用类似于一个string或一个[]byte。我们可以调用len它，我们可以索引它，我们可以将索引操作的结果与字节值进行比较。

要进行编译，type参数T本身需要一个类型。它是一个元类型，但是因为我们有时需要描述多个相关类型，并且因为它描述了泛型函数的实现与其调用者之间的关系，所以我们实际上调用T了契约的类型。合约在这里命名Sequence。它出现在类型参数列表之后。

这是为此示例定义Sequence契约的方式。

```go
contract Sequence(T) {
    T string, []byte
}
```
这很简单，因为这是一个简单的例子：type参数 T可以是string或[]byte。这contract可能是一个新的关键字，或在包范围内识别的特殊标识符; 请参阅设计草案了解详情。

任何记得我们在[Gophercon 2018上展示过的设计](https://github.com/golang/proposal/blob/4a530dae40977758e47b78fae349d8e5f86a6c0a/design/go2draft-contracts.md)的人 都会发现这种签订合约的方式要简单得多。我们得到了很多关于合约过于复杂的早期设计的反馈，我们已经尝试将其考虑在内。新合约的编写，阅读和理解都要简单得多。

它们允许您指定类型参数的基础类型，和/或列出类型参数的方法。它们还可以让您描述不同类型参数之间的关系。

## 与方法签订合约

这是另一个简单的例子，它使用String方法返回[]string所有元素的字符串表示形式s。

```go
func ToStrings (type E Stringer) (s []E) []string {
    r := make([]string, len(s))
    for i, v := range s {
        r[i] = v.String()
    }
    return r
}
```
它非常简单：遍历切片，String 在每个元素上调用方法，然后返回结果字符串的切片。

此函数要求元素类型实现该String 方法。字符串合约确保了这一点。

```go
contract Stringer(T) {
    T String() string
}
```
合约只是说T必须实施该String 方法。

您可能会注意到此合约看起来像fmt.Stringer 接口，因此值得指出ToStrings函数的参数 不是一个分支fmt.Stringer。它是元素类型实现的一些元素类型的片段 fmt.Stringer。元素类型切片的内存表示和fmt.Stringer 切片通常是不同的，Go不支持它们之间的直接转换。所以即使fmt.Stringer存在，这是值得写的。

## 有多种类型的合约

以下是具有多个类型参数的合约示例。

```go
type Graph (type Node, Edge G) struct { ... }

contract G(Node, Edge) {
    Node Edges() []Edge
    Edge Nodes() (from Node, to Node)
}

func New (type Node, Edge G) (nodes []Node) *Graph(Node, Edge) {
    ...
}

func (g *Graph(Node, Edge)) ShortestPath(from, to Node) []Edge {
    ...
}
```

这里我们描述一个由Node和Edge构建的Graph。我们不需要Graph的特定数据结构。相反，我们说Node类型必须有一个Edges 方法，返回连接到的Edge的列表Node。而且Edge类型必须有一个Nodes返回两个方法 Nodes，该Edge所连接。

我已经跳过了实现，但是这显示了一个New返回a 的 函数Graph的签名，以及一个ShortestPath方法的签名 Graph。

这里的重要内容是合约不仅仅是一种类型。它可以描述两种或更多种类型之间的关系。

## 有序类型

关于Go的一个令人惊讶的常见抱怨是它没有Min函数。同样也没有Max函数。这是因为一个有用的Min函数应该适用于任何有序类型，这意味着它必须是通用的。

虽然Min编写自己非常简单，但任何有用的泛型实现都应该让我们将它添加到标准库中。这就是我们的设计。

```go
func Min (type T Ordered) (a, b T) T {
    if a < b {
        return a
    }
    return b
}
```

该Ordered合约上说T具有类型是有序类型，这意味着它支持像小于，大于，等运算。

```go
contract Ordered(T) {
    T int, int8, int16, int32, int64,
        uint, uint8, uint16, uint32, uint64, uintptr,
        float32, float64,
        string
}
```
该Ordered合约仅仅是所有由语言定义的有序类型的列表。此合约接受任何列出的类型，或其基础类型为其中一种类型的任何命名类型。基本上，您可以使用任何可以使用小于运算符的类型。

事实证明，简单地枚举支持小于运算符的类型比发明一个适用于所有运算符的新表示法要容易得多。毕竟，在Go中，只有内置类型支持运算符。

这种方法可以用于任何运算符，或者更一般地，用于编写旨在与内置类型一起使用的任何通用函数的契约。它允许泛型函数的编写者清楚地指定函数应该与之一起使用的类型集。它允许通用函数的调用者清楚地看到该函数是否适用于所使用的类型。

实际上，这份合约可能会进入标准库。所以Min函数（可能也会在某个标准库中）看起来就像这样。这里我们只是提到包中定义的Ordered合约。

```go
func Min (type T contracts.Ordered) (a, b T) T {
    if a < b {
        return a
    }
    return b
}
```

## 通用数据结构

最后，让我们看一个简单的通用数据结构，一个二叉树。在此示例中，树具有比较功能，因此对元素类型没有要求。

```go
type Tree (type E) struct {
    root    *node(E)
    compare func(E, E) int
}

type node (type E) struct {
    val         E
    left, right *node(E)
}
```
以下是如何创建新的二叉树。比较函数传递给New函数。

```go
func New (type E) (cmp func(E, E) int) *Tree(E) {
    return &Tree(E){compare: cmp}
}
```
未导出的方法返回一个指向持有v的槽的指针，或指向树应该去的位置。

```go
func (t *Tree(E)) find(v E) **node(E) {
    pn := &t.root
    for *pn != nil {
        switch cmp := t.compare(v, (*pn).val); {
        case cmp < 0:
            pn = &(*pn).left
        case cmp > 0:
            pn = &(*pn).right
        default:
            return pn
        }
    }
    return pn
}
```
这里的细节并不重要，特别是因为我没有测试过这段代码。我只想展示编写简单通用数据结构的样子。

这是用于测试树是否包含值的代码。

```go
func (t *Tree(E)) Contains(v E) bool {
    return *t.find(e) != nil
}
```
这是插入新值的代码。

```go
func (t *Tree(E)) Insert(v E) bool {
    pn := t.find(v)
    if *pn != nil {
        return false
    }
    *pn = &node(E){val: v}
    return true
}
```

注意类型参数E的类型node。这就是编写通用数据结构的样子。正如您所看到的，它看起来像编写普通的Go代码，除了在这里和那里散布一些类型的参数。

使用树很简单。

```go
var intTree = tree.New(func(a, b int) int { return a - b })

func InsertAndCheck(v int) {
    intTree.Insert(v)
    if !intTree.Contains(v) {
        log.Fatalf("%d not found after insertion", v)
    }
}
```
这是应该的。编写通用数据结构要困难一些，因为您经常需要为支持类型显式写出类型参数，但尽可能使用一个与使用普通的非通用数据结构没有什么不同。

## 下一步

我们正在开展实际实施，以便我们尝试这种设计。能够在实践中尝试设计非常重要，以确保我们可以编写我们想要编写的程序类型。它没有像我们希望的那样快，但我们将在这些实现可用时发送更多详细信息。

Robert Griesemer编写了一个[初步的CL](https://golang.org/cl/187317)，修改了go/types包。这允许测试使用泛型和合约的代码是否可以键入检查。它现在还不完整，但它主要适用于单个包，我们将继续努力。

我们希望人们对这个和未来的实现做的是尝试编写和使用通用代码，看看会发生什么。我们希望确保人们可以编写他们需要的代码，并且他们可以按预期使用它。当然，一开始并不会所有的都行得通，随着我们不断的探索，我们可能不得不改变一些东西。而且，要清楚，我们对语义的反馈比对语法的细节更感兴趣。

我要感谢所有对早期设计发表评论的人，以及所有讨论过Go中泛型的人。我们已经阅读了所有评论，我们非常感谢人们为此付出的努力。如果没有那项工作，我们就不会成为今天的所在。

我们的目标是实现一种设计，使我能够编写我今天讨论的通用代码，而不会使语言过于复杂或不再使用Go。我们希望这个设计是朝着这个目标迈出的一步，我们希望在我们学习的过程中继续调整它，从我们的经验和你的经验，什么有效，什么无效。如果我们确实达到了这个目标，那么我们就可以为Go的未来版本提供一些建议。

作者：Ian Lance Taylor

原文：https://blog.golang.org/why-generics

译文：https://github.com/llgoer/go-generics