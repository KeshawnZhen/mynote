# UTF
Unicode的最初目标，是用1个16位的编码来为超过65000字符提供映射。但它不能覆盖全部历史上的文字，也不能解决传输的问题 (implantation head-ache's)，尤其在那些基于网络的应用中。已有的软件必须做大量的工作来传输16位的数据。 

因此，Unicode用一些基本的保留字符制定了三套编码方式。它们分别是UTF-8,UTF-16和UTF-32。在UTF－8中，字符是以8位序列来编码的，用一个或几个字节来表示一个字符。这种方式的最大好处，是UTF－8保留了ASCII字符的编码做为它的一部分，例如，在 UTF－8和ASCII中，“A”的编码都是0x41. 

UTF-16和UTF-32分别是Unicode的16位和32位编码方式。考虑到最初的目的，通常说的Unicode就是指UTF-16。Unicdoe相关的技术介绍参见http://www.unicode.org/unicode/standard/principles.html.

UTF-8是变长编码，每个Unicode代码点按照不同范围，可以有1-3字节的不同长度。UTF-16长度相对固定，只要不处理大于\U200000范围的字符，每个Unicode代码点使用16位即2字节表示，超出部分使用两个UTF-16即4字节表示。按照高低位字节顺序，又分为UTF-16BE/UTF-16LE。UTF-32长度始终固定，每个Unicode代码点使用32位即4字节表示。按照高低位字节顺序，又分为UTF-32BE/UTF-32LE。


## 优点：
UTF-8
1. 兼容 ASCII
2. 能适应许多 C 库中的 \0 结尾惯例
3. 没有字节序问题
4. 良好的多语种支持（相对 GBK 等跟语种绑定的编码方式）
5. 以英文和西文符号比较多的场景下（例如 HTML/XML），编码较短
6. 由于是变长，字符空间足够大，未来 Unicode 新标准收录更多字符，UTF-8 也能妥妥的兼容，因此不会再出现 UTF-16 那样的尴尬
7. 不存在大小端字节序问题，信息交换时非常便捷
8. 容错性高，局部的字节错误（丢失、增加、改变）不会导致连锁性的错误，因为 UTF-8 的字符边界很容易检测出来，这是一个巨大的优点

UTF-16
1. 在各大流行的操作系统中被广泛使用（windows、IOS、OSX等）
2. 在计算字符串长度、执行索引操作时速度很快。（但是UTF-16也是变长的，Unicode扩展到9万多以后，也要通过变长来支持了。）

UTF-32
1. 定长编码，utf32 表示任何字符都用 4 字节，读到内存中是个均匀的整形数组，于是我们可以很方便地随机访问任何一个字符
2. 由于是定长，索引比变长的要快，你想访问一个字符串中的第 n 个字符，utf32 直接偏移 n 个整形距离即可，utf8 得从第一个字节一个字一个字地往后检索

## 缺点：
UTF-8
1. 对于英文字符，UTF-8 很棒，因为它和 ASCII 一样，一个字符只占一个字节，没有任何额外的存储负担；但是对于中文字符，UTF-8 过于冗余，一个字符要占用 3 个字节，存储和传输的效率不但没有提升，反而下降了。
2. 变长字节表示带来的效率问题——大家对 UTF-8 疑虑重重的一个问题就是在于其因为是变长字节表示，因此无论是计算字符数，还是执行索引操作效率都不高。为了解决这个问题，常常会考虑把 UTF-8 先转换为 UTF-16 或者 UTF-32 后再操作，操作完毕后再转换回去。而这显然是一种性能负担。

UTF-16
1. UTF-16 能表示的字符数有 6 万多，看起来很多，但是实际上目前 Unicode 5.0 收录的字符已经达到 99024 个字符，早已超过 UTF-16 的存储范围；这直接导致 UTF-16 地位颇为尴尬
2. UTF-16 存在大小端字节序问题，这个问题在进行信息交换时特别突出——如果字节序未协商好，将导致乱码；如果协商好，但是双方一个采用大端一个采用小端，则必然有一方要进行大小端转换，性能损失不可避免（大小端问题其实不像看起来那么简单，有时会涉及硬件、操作系统、上层软件多个层次，可能会进行多次转换）。
3. 容错性低有时候也是一大问题。局部的字节错误，特别是丢失或增加可能导致所有后续字符全部错乱，错乱后要想恢复，可能很简单，也可能会非常困难。

# BOM
字节顺序标记(byte order mark, BOM)是一个Unicode字符，U+FEFF字节顺序标记(byte order mark, BOM)，它在文本流开始时作为一个魔数出现，可以向读取文本的程序提供以下信息:
1. 文本流的字节顺序（大端小端）;
2. 文本流的编码是Unicode，这一点非常可靠;
3. 编码文本流的Unicode编码方式。

BOM使用是可选的。它的存在干扰了软件对UTF-8的使用，这些软件在文件开始时不期望使用非ascii字节，但可以处理文本流。

Unicode可以以8位、16位或32位整数为单位进行编码。对于16位和32位表示，接收任意源文本的计算机需要知道整数编码的字节顺序。BOM与文档的其他部分采用相同的格式进行编码，如果交换了它的字节，它就成为一个非字符Unicode编码点。因此，访问文本的进程可以检查前几个字节来确定endianess，而不需要文本流本身之外的一些契约或元数据。一般情况下，接收计算机会根据需要将字节交换为自己的字节，不再需要BOM进行处理。

BOM的字节序列因Unicode编码而异(包括Unicode标准之外的编码，如UTF-7)，而且在存储在其他编码中的文本流的开始位置不太可能出现任何序列。因此，在文本流的开始处放置一个经过编码的BOM可以指示文本是Unicode，并识别使用的编码方案。BOM字符的这种用法称为“Unicode签名”

 [BOM](https://en.wikipedia.org/wiki/Byte_order_mark)

# GBK

## 1  GB2312-80
GB 2312 或 GB 2312-80 是中国国家标准简体中文字符集，全称《信息交换用汉字编码字符集·基本集》，又称 GB 0，由中国国家标准总局发布，1981 年 5 月 1 日实施。GB 2312 编码通行于中国大陆；新加坡等地也采用此编码。中国大陆几乎所有的中文系统和国际化的软件都支持 GB 2312。

GB 2312 标准共收录 6763 个汉字，其中一级汉字 3755 个，二级汉字 3008 个；同时收录了包括拉丁字母、希腊字母、日文平假名及片假名字母、俄语西里尔字母在内的 682 个字符。

1. GB 2312 的出现，基本满足了汉字的计算机处理需要，它所收录的汉字已经覆盖中国大陆99.75% 的使用频率。
2. 对于人名、古汉语等方面出现的罕用字，GB 2312 不能处理，这导致了后来 GBK 及 GB 18030 汉字字符集的出现。

GB 2312 对任意一个图形字符都采用两个字节表示，并对所收汉字进行了“分区”处理，每区含有 94 个汉字／符号，分别对应第一字节和第二字节。这种表示方式也称为区位码。
1. 01-09 区为特殊符号。
2. 16-55 区为一级汉字，按拼音排序。
3. 56-87 区为二级汉字，按部首／笔画排序。

10-15 区及 88-94 区则未有编码。GB 2312 的编码范围为 2121H-777EH，与 ASCII 有重叠，通行方法是将 GB 码两个字节的最高位置 1 以示区别。

## 2  GBK
GBK 即汉字内码扩展规范

GBK 共收入 21886 个汉字和图形符号，包括：
- GB 2312 中的全部汉字、非汉字符号。
- BIG5 中的全部汉字。
- 与 ISO 10646 相应的国家标准 GB 13000 中的其它 CJK 汉字，以上合计 20902 个汉字。
- 其它汉字、部首、符号，共计 984 个。
GBK 向下与 GB 2312 完全兼容，向上支持 ISO 10646 国际标准，在前者向后者过渡过程中起到的承上启下的作用。
GBK 采用双字节表示，总体编码范围为 8140-FEFE 之间，首字节在 81-FE 之间，尾字节在 40-FE 之间，剔除 XX7F 一条线。GBK 编码区分三部分：
- 汉字区　包括 
 GBK/2：OXBOA1-F7FE, 收录 GB 2312 汉字 6763 个，按原序排列； GBK/3：OX8140-AOFE，收录 CJK 汉字 6080 个；  
 GBK/4：OXAA40-FEAO，收录 CJK 汉字和增补的汉字 8160 个。
- 图形符号区　包括  
GBK/1：OXA1A1-A9FE，除 GB 2312 的符号外，还增补了其它符号        
GBK/5：OXA840-A9AO，扩除非汉字区。
- 用户自定义区 
GBK 区域中的空白区，用户可以自己定义字符。


## 3 GB18030
GB 18030，全称：国家标准 GB 18030-2005《信息技术中文编码字符集》，是中华人民共和国现时最新的内码字集，是 GB 18030-2000《信息技术信息交换用汉字编码字符集基本集的扩充》的修订版。
GB 18030 与 GB 2312-1980 和 GBK 兼容，共收录汉字70244个。
- 与 UTF-8 相同，采用多字节编码，每个字可以由 1 个、2 个或 4 个字节组成。
- 编码空间庞大，最多可定义 161 万个字符。
- 支持中国国内少数民族的文字，不需要动用造字区。
- 汉字收录范围包含繁体汉字以及日韩汉字

GB 18030 编码是一二四字节变长编码。
- 单字节，其值从 0 到 0x7F，与 ASCII 编码兼容。
- 双字节，第一个字节的值从 0x81 到 0xFE，第二个字节的值从 0x40 到 0xFE（不包括0x7F），与 GBK 标准兼容。
- 四字节，第一个字节的值从 0x81 到 0xFE，第二个字节的值从 0x30 到 0x39，第三个字节从0x81 到 0xFE，第四个字节从 0x30 到 0x39。

# UTF和GBK的区别
UTF-8编码则是用以解决国际上字符的一种多字节编码，它对英文使用8位（即一个字节），中文使用24位（三个字节）来编码。对于英文字符较多的论坛则用UTF-8节省空间。

GBK包含全部中文字符；UTF-8则包含全世界所有国家需要用到的字符。

GBK是在国家标准GB2312基础上扩容后兼容GB2312的标准

UTF-8编码的文字可以在各国各种支持UTF8字符集的浏览器上显示。比如，如果是UTF8编码，则在外国人的英文IE上也能显示中文，而无需他们下载IE的中文语言支持包。 所以，对于英文比较多的论坛 ，使用GBK则每个字符占用2个字节，而使用UTF-8英文却只占一个字节。

UTF8是国际编码，它的通用性比较好，外国人也可以浏览论坛，GBK是国家编码，通用性比UTF8差。数据库中使用UTF8编码格式存储占用空间比GBK大



# Java中编码
```
public static void main(String[] args) {
    System.out.println(System.getProperty("file.encoding"));
    try {
        String str = new String("hhhh ty智障%shfu摸淑芬十分uif内服NSF黑");
        // 1.以GBK编码方式获取str的字节数组，再用String有参构造函数构造字符串
        System.out.println(new String(str.getBytes("GBK")));
        // 2.以UTF-8编码方式获取str的字节数组，再以默认编码构造字符串
        System.out.println(new String(str.getBytes("UTF-8")));
    } catch (UnsupportedEncodingException e) {
        e.printStackTrace();
    }
}
```
.java文件用UTF-8编码保存时的输出
```
UTF-8
hhhh ty����%shfu�����ʮ��uif�ڷ�NSF��
hhhh ty智障%shfu摸淑芬十分uif内服NSF黑
```
看到utf8的结果是正确的，但gbk是错误的

.java文件用GBK编码保存时的输出
```
UTF-8
hhhh ty????%shfu????????uif???NSF??
hhhh ty����%shfu�����ʮ��uif�ڷ�NSF��
```
这个输出是什么鬼，gbk怎么也不对？在sun.nio.cs的包下就没找到gbk的类，是不是这个原因我暂时还不知道

看看源码吧
String.java
```
public byte[] getBytes(String charsetName)
        throws UnsupportedEncodingException {
    if (charsetName == null) throw new NullPointerException();
    return StringCoding.encode(charsetName, value, 0, value.length);
}
```
StringCoding.java
```
static byte[] encode(String charsetName, char[] ca, int off, int len)
    throws UnsupportedEncodingException
{
    StringEncoder se = deref(encoder);
    //默认使用ISO-8859-1
    String csn = (charsetName == null) ? "ISO-8859-1" : charsetName;
    if ((se == null) || !(csn.equals(se.requestedCharsetName())
                            || csn.equals(se.charsetName()))) {
        se = null;
        try {
            Charset cs = lookupCharset(csn);
            if (cs != null)
                se = new StringEncoder(cs, csn);
        } catch (IllegalCharsetNameException x) {}
        if (se == null)
            throw new UnsupportedEncodingException (csn);
        set(encoder, se);
    }
    return se.encode(ca, off, len);
}
```

java.nio.charset.Charset
```
public static Charset defaultCharset() {
    if (defaultCharset == null) {
        synchronized (Charset.class) {
            String csn = AccessController.doPrivileged(
                new GetPropertyAction("file.encoding"));
            Charset cs = lookup(csn);
            if (cs != null)
                defaultCharset = cs;
            else
                defaultCharset = forName("UTF-8");
        }
    }
    return defaultCharset;
}
```

总结一下：
首先，Java源码文件。这些文件可以是任意字符编码的。然后在Java的Class文件里存储的字符串是UTF-8编码的。从Java源码文件到Java Class文件，中间会经过Java源码编译器（例如javac或ECJ）的编译。
也就是说，是Java源码编译器负责将Java源码文件的编码转换为最终的UTF-8。导致乱码的不是Java源码编译器的“编码”（写出UTF-8）的过程，而是“解码”（读入Java源码内容）的过程。

以javac为例，它有一个参数可以指定输入的Java源码文件的编码：
>-encoding encodingSet the source file encoding name, such as EUC-JP and UTF-8. If -encoding is not specified, the platform default converter is used.

关键在于“如果不指定encoding，则使用平台默认的转换器”。在简体中文的Windows上，平台默认编码会是GBK，那么javac就会默认假定输入的Java源码文件是以GBK编码的。javac能够正确读取文件内容并将其中的字符串以UTF-8输出到Class文件里，就跟自己写个程序以GBK读文件以UTF-8写文件一样。如果实际输入的确实是GBK编码（GBK兼容ASCII编码）的文件，那么一切都会正常。但如果实际输入的是别的编码的文件，例如超过了ASCII范围的UTF-8，那javac读进来的内容就会出问题，就“乱码”了。


