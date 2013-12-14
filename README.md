#code LESS, sleep MORE
今天，也就是2013年12月14日，在给部门培训完《重构到模式》后，随意翻看微博，看到dreamhead在晒他在[github](https://github.com/dreamhead)的连续300天提交代码。然后我就到他的github上随意翻了一下，看看他有哪些项目，就随意浏览到了他的[ugly-code](https://github.com/dreamhead/ugly-code)，觉得这种连着案例的模式挺不错，只是没有长期更新，我先把他的内容抄袭过来，后面再添加我自己收集的一些典型案例。
前14章直接先抄过来。

##Table of Contents
- [1. 使用直接的布尔表达式](#使用直接的布尔表达式)
- [2. 让判断条件做真正的选择](#让判断条件做真正的选择)
- [3. 判断条件不允许多于3个条件](#判断条件不允许多于3个条件)
- [4. switch的裹脚](#switch的裹脚)
- [5. 似曾相似，似是而非的代码](#似曾相似，似是而非的代码)
- [6. 声明和使用应尽量接近](#代码的声明和使用应尽量接近)


## 使用直接的布尔表达式(From )
看到下面这段代码，你有想法么？

```java
if (db.next() == true) {
    return true;   
} else {
    return false;
}
```

“我靠”，有的人会想，“怎么写得这么笨啊”！但是，请放心，绝对会有人这么想，挺好的，实现功能了。这并非我臆造出的代码，而是从一个真实的codebase上找到。

成为一个咨询师之后，我有机会在不同的项目中穿梭。同客户合作的过程中，我经常干的一件事是：code diff。也就是用源码管理工具的diff功能把当天全部修改拿出来，从编码的角度来分析代码写得怎么样。

因为这个工作，我看到了许多不同人编写的代码，我的编码底线不断受到挑战。许多东西，我以为是常识，但实际上不为许多人所知，比如上面那段代码。

重构后的代码会是这样：

```java
return db.next();
```


##让判断条件做真正的选择
诸位看官，上代码：

```c++
if (0 == iRetCode) {
    this->SendPeerMsg("000", "Process Success", outRSet);
} else {
    this->SendPeerMsg("000", "Process Failure", outRSet);
}
```

乍一看，这段代码还算比较简短。那下面这段呢？

```c++
if (!strcmp(pRec->GetRecType(), PUB_RECTYPE::G_INSTALL)) {
    CommDM.jkjtVPDNResOperChangGroupInfo(
      const_cast(CommDM.GetProdAttrVal("vpdnIPAddress",
      &(pGroupSubs->m_ProdAttr))),
      true);
} else {
    CommDM.jkjtVPDNResOperChangGroupInfo(
      const_cast(CommDM.GetProdAttrVal("vpdnIPAddress",
      &(pGroupSubs->m_ProdAttr))),
      false);
}
```

看出来问题了吗？经过仔细的对比，我们发现，对于如此华丽的代码，if/else的执行语句真正的差异只在于一个参数。第一段代码，二者的差异只是发送的消息，第二段代码，差异在于最后那个参数。

看破这个差异之后，新的写法就呼之欲出了，以第一段代码为例：

```c++
const char* msg = (0 == iRetCode ? "Process Success" : "Process Failure");
this->SendPeerMsg("000", msg, outRSet);
```

为了节省篇幅，我选择了三元表达式。我知道，很多人不是那么喜欢它。如果if/else依旧是你的大爱，勇敢追求去吧！

由这段代码调整过程，我们得出一个简单的规则：

> **让判断条件做真正的选择**。

这里判断条件真正判断的内容是消息的内容，而不是消息发送的过程。经过我们的调整，得到消息内容和和发送消息的过程严格分离开来。

消除了代码中的冗余，代码也更容易理解，同时，给未来留出了可扩展性。如果将来iRetCode还有更多的情形，我们只要在消息获取的时候进行调整就好了。当然，封装成一个函数是一个更好的选择，这样代码就变成了：

```c++
this->SendPeerMsg("000", peerMsgFromRetCode(iRetCode), outRSet);
```

至于第二段代码的调整，留给你练手了。

这样丑陋的代码是如何从众多代码中脱颖而出的呢？很简单，只要看到，if/else两个执行块里面的内容相差无几，需要我们人工比字符寻找差异，恭喜你，你找到它了。

##判断条件不允许多于3个条件
这里有一个长长的“故事”：

```c++
if (strcmp(rec.type, "PreDropGroupSubs") == 0
    || strcmp(rec.type, "StopUserGroupSubsCancel") == 0
    || strcmp(rec.type, "QFStopUserGroupSubs") == 0
    || strcmp(rec.type, "QFStopUserGroupSubsCancel") == 0
    || strcmp(rec.type, "QZStopUserGroupSubs") == 0
    || strcmp(rec.type, "QZStopUserGroupSubsCancel") == 0
    || strcmp(rec.type, "SQStopUserGroupSubs") == 0
    || strcmp(rec.type, "SQStopUserGroupSubsCancel") == 0
    || strcmp(rec.type, "StopUseGroupSubs") == 0
    || strcmp(rec.type, "PreDropGroupSubsCancel") == 0)
```

之所以注意到它，因为最后两个条件是最新修改里面加入的，换句话说，这不是一次写就的代码。单就这一次而言，只改了两行，这是可以接受的。但这是遗留代码。每次可能只改了一两行，通常我们会不只一次踏入这片土地。经年累月，代码成了这个样子。

这并非我接触过的最长的判断条件，这种代码极大的开拓了我的视野。现在的我，即便面对的是一屏无法容纳的条件，也可以坦然面对了，虽然显示器越来越大。

其实，如果这个判断条件是这个函数里仅有的东西，我也就忍了。遗憾的是，大多数情况下，这只不过是一个更大函数中的一小段而已。

为了让这段代码可以接受一些，我们不妨稍做封装：

```c++
bool shouldExecute(Record& rec) {
    return (strcmp(rec.type, "PreDropGroupSubs") == 0
      || strcmp(rec.type, "StopUserGroupSubsCancel") == 0
      || strcmp(rec.type, "QFStopUserGroupSubs") == 0
      || strcmp(rec.type, "QFStopUserGroupSubsCancel") == 0
      || strcmp(rec.type, "QZStopUserGroupSubs") == 0
      || strcmp(rec.type, "QZStopUserGroupSubsCancel") == 0
      || strcmp(rec.type, "SQStopUserGroupSubs") == 0
      || strcmp(rec.type, "SQStopUserGroupSubsCancel") == 0
      || strcmp(rec.type, "StopUseGroupSubs") == 0
      || strcmp(rec.type, "PreDropGroupSubsCancel") == 0);
}

if (shouldExecute(rec)) {
    // ...
}
```

现在，虽然条件依然还是很多，但和原来庞大的函数相比，至少它已经被控制在一个相对较小的函数里了。更重要的是，通过函数名，我们终于有机会说出这段代码判断的是什么了。

提取函数把这段代码混乱的条件分离开来，它还是可以继续改进的。比如，我们把判断的条件进一步提取：

```c++
bool shouldExecute(Record& rec) {
  static const char* execute_types[] = {
    "PreDropGroupSubs",
    "StopUserGroupSubsCancel",
    "QFStopUserGroupSubs",
    "QFStopUserGroupSubsCancel",
    "QZStopUserGroupSubs",
    "QZStopUserGroupSubsCancel",
    "SQStopUserGroupSubs",
    "SQStopUserGroupSubsCancel",
    "StopUseGroupSubs",
    "PreDropGroupSubsCancel"
  };

  for (int i = 0, ii = ARRAY_SIZE(execute_types); i < ii; ++i) {
    if (strcmp(rec.type, execute_type[i]) == 0) return true;
  }
  
  return false;
}
```

这样的话，再加一个新的type，只要在数组中增加一个新的元素即可。如果我们有兴趣的话，还可以进一步对这段代码进行封装，把这个type列表变成声明式，进一步提高代码的可读性。

发现这种代码很容易，只要看到在长长的判断条件，就是它了。要限制这种代码的存在，我们只要以设定一个简单的规则：

> **判断条件里面不允许多于3个条件的组合**。

虽然通过不断调整，这段代码已经不同于之前，但它依然不是我们心目中的理想代码。出现这种代码，往往意味背后有更严重的设计问题。不过，它并不是这里讨论的内容，这里的讨论就到此为止吧！

##switch的裹脚
又见switch：

```c++
switch(firstChar) {
  case 'N':
    nextFirstChar = 'O';
    break;
  case 'O':
    nextFirstChar = 'P';
    break;
  case 'P':
    nextFirstChar = 'Q';
    break;
  case 'Q':
    nextFirstChar = 'R';
    break;
  case 'R':
    nextFirstChar = 'S';
    break;
  case 'S':
    nextFirstChar = 'T';
    break;
  case 'T':
    throw new BusinessException();
  default:
}
```

出于多年编程养成的条件反射，我对于switch总会给予更多的关照。研习面向对象编程之后，看见switch就会想到多态，遗憾的是，这段代码和多态没什么关系。仔细阅读这段代码，我找出了其中的规律，nextFirstChar就是firstChar的下一个字符。于是，我改写了这段代码：

```c++
switch(firstChar) {
  case 'N':
  case 'O':
  case 'P':
  case 'Q':
  case 'R':
    nextFirstChar = firstChar + 1;
    break;
  case 'T':
    throw new BusinessException();
  default:
}
```

现在，至少看起来，这段代码已经比原来短了不少。当然这么做基于一个前提，也就是这些字母编码的顺序确确实实连续的。从理论上说，开始那段代码适用性更强。但在实际开发中，我们碰到字母不连续编码的概率趋近于0。

但这段代码究竟是如何产生的呢？我开始研读上下文，原来这段代码是用当前ID产生下一个ID的，比如当前是N0000，下一个就是N0001。如果数字满了，就改变字母，比如当前ID是R9999，下一个就是T0000。在这里，字母也就相当于一位数字，根据情况进行进位，所以有了这段代码。

代码上的注释告诉我，字母的序列只有从N到T，根据这个提示，我再次改写了这段代码：

```c++
if (firstChar >= 'N' && firstChar <= 'S') {
    nextFirstChar = firstChar + 1;
} else {
    throw new BusinessException();();
}
```

这里统一处理了字母为T和default的情形，严格说来，这和原有代码并不完全等价。但这是了解了需求后做出的决定，换句话说，原有代码在这里的处理中存在漏洞。

修改这段代码，只是运用了非常简单的编程技巧。遗憾的是，即便如此简单的编程技巧，也不是所有开发人员都驾轻就熟的，很多人更习惯于“平铺直叙”。这种直白造就了代码中的许多鸿篇巨制。我听过不少“编程是体力活”的抱怨，不过，能把写程序干成体力活，也着实不值得同情。写程序，不动脑子，不体力才怪。

无论何时何地，只要switch出现在眼前，请提高警惕，那里多半有坑。

## 似曾相似，似是而非的代码
这是一个找茬的游戏，下面三段代码的差别在哪：

```c++
if ( 1 == SignRunToInsert)    {
    RetList->Insert(i, NewCatalog);
} else {
    RetList->Add(NewCatalog);
}

if ( 1 == SignRunToInsert)    {
    RetList->Insert(m, NewCatalog);
} else {
    RetList->Add(NewCatalog);
}

if ( 1 == SignRunToInsert)    {
    RetList->Insert(j, NewPrivNode);
} else {
    RetList->Add(NewPrivNode);
}
```

答案时间：除了用到变量之外，完全相同。我想说的是，这是我从一个文件的一次diff中看到的。

不妨设想一下修改这些代码时的情形：费尽九牛二虎之力，我终于找到该在哪改动代码，改了。作为一个有职业操守的程序员，我知道别的地方也需要类似的修改。于是，趁人不备，把刚做修改拷贝了一份，放到另外需要修改的地方。修改了几个变量，编译通过了。世界应该就此清净，至少问题解决了。

好吧！虽然这个程序员有职业操守的程序员，却缺少了些职业技能，至少在挥舞“拷贝粘贴”时，他没有嗅到散发出的臭味。

只要意识到坏味道，修改是件很容易的事，提出一个新函数即可：

```c++
void AddNode(List& RetList, int SignRunToInsert, int Pos, Node& Node) {
    if ( 1 == SignRunToInsert)    {
      RetList->Insert(Pos, Node);
    } else {
      RetList->Add(Node);
    }
}
```

于是，原来那三段代码变成了三个调用：

```c++
AddNode(RetList, SignRunToInsert, i, NewCatalog);
AddNode(RetList, SignRunToInsert, m, NewCatalog);
AddNode(RetList, SignRunToInsert, j, NewPrivNode);
```

当然，这种修改只是一个局部的微调，如果有更多的上下文信息，我们可以做得更好。

重复，是最为常见的坏味道。上面这种重复实际上是非常容易发现的，也是很容易修改。但所有这一切的前提是，发现坏味道。

长时间生活在这种代码里面，我们会对坏味道失去嗅觉。更可怕的是，一个初来乍到的嗅觉尚灵敏的人意识到这个问题，那些失去嗅觉的人却告诫他，别乱动，这挺好。

趁嗅觉尚在，请坚持代码正义。

#代码的声明和使用应尽量接近
这是一段长长的C++代码，我的问题是：relaPri、relaSec和 scoutBySec这三个变量在哪里用到了？

```c++
void DealForServiceA(const char *oprCode, const char *subID, const char *oID, XList *callCicsList) {
    XString relaPri("NULL");
    XString relaSec("NULL");
    XString scoutBySec("0");
    XList *tempList = new XList ;
    callCicsList->Add(tempList);
    tempList->Add(new XString(oprCode));
    tempList->Add(new XString(oID));
    XString *psTelNum = new XString;
    tempList->Add(psTelNum);
    GetServnumberBySubsID(subID, *psTelNum);   
    tempList->Add(new XString(relaPri.table { font-size: 10pt;}c_str()));
    tempList->Add(new XString(relaSec.c_str()));
    tempList->Add(new XString(scoutBySec.c_str()));
}
```

经过认真仔细的查看，或是使用传说的中“查找”功能，我们发现上面提到的那三个变量只在最后用了一下。

不知道你是否注意到，我在最初特意强调了一下这是C++代码。这意味着，变量可以随用随声明，而不必像传统的C程序那样，只能在函数的开头把函数内部用到的变量一口气声明。 那么 ，我们就让声明和使用团聚吧！

```c++
XString relaPri("NULL");
tempList->Add(new XString(relaPri.c_str()));
XString relaSec("NULL");
tempList->Add(new XString(relaSec.c_str()));
XString scoutBySec("0");
tempList->Add(new XString(scoutBySec.c_str()));
```

当声明和使用走到一起，我们的观察就有了新的视角，其实，这几个变量完全是可以不声明的，于是，代码再进一步：

```c++
tempList->Add(new XString("NULL"));
tempList->Add(new XString("NULL"));
tempList->Add(new XString("0"));
```

看到这里，我们就可以看出原来的做法到底有多么浪费：浪费时间给变量起名字——我们都知道，起个好名字不容易，也浪费了时间在执行上，修改前的代码创建了两个XString对象，而修改后，只创建了一个对象。

或许，你会觉得，有个变量会让我们了解这里实际上填加的内容到底是什么。不过，也许一个好的函数命名才是更好的选择，比如 addRelaPri。这个疑问会揭示出这段代码存在另外一个问题，直接使用基本的数据结构而没有进行封装。不过，这不是这里讨论的目标，就到此打住吧！

根据这段代码的调整，我们得出一条规则：

> **代码的声明和使用应尽量接近**。

有的C程序员会暗自念叨，这个要求对C程序来说，简直太不合情理了。好吧！我承认，从语言的角度来说，是这样的。但是，我们需要仔细想想，为什么对于C语言来说，变量的声明和使用会距离遥远。通常，遥远的背后意味着硕大的函数，这才是让声明和使用天各一方的重要原因。

在干净代码的世界里，大函数永远是不受欢迎的。为了让声明和使用尽早团聚，请把函数写小。

