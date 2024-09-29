---
layout: post
title:  "xjar支持springboot3分析"
date:   2024-09-29 10:29:20 +0800
categories:
      - java
      - 加密
tags:
      - jar包加密
---

最近公司需要将项目部署在第三方服务器，于是就有了jar包加密的需求，了解了下目前加密方案现况如下:

1. 混淆方案，就是在代码中添加大量伪代码，以便隐藏业务代码
2. 加密方案，将jar包中的所有class加密，在运行时通过自定义classloader进行解密

目前有的所有加密方案基本思路都跟上面大差不差，在了解了一圈决定使用了xjar这个开源项目。它的实现思路就是方案2.

打开github首页，xjar项目文档还是不错的，clone下来，跟着文档上手，很容易就测试并通过了我的测试项目，接着便推给其他同事使用了。

好景不长，下午就有同事找我，说他的项目加密后不能成功运行。我去看了下，加密操作上没什么问题，但是就是加密包不能成功运行。
报错如下：
```
错误: 找不到或无法加载主类 null
原因: java.lang.ClassNotFoundException: null
panic: exit status 1
```
了解了下，同事那边使用了springboot3，而我测试项目是springboot2。 难道不支持springboot3? 我心里想到。

先简单看了下加密的jar包目录结构，很容易就发现以下问题：
1. jar包中MANIFEST.MF文件中, Main-Class属性没有值
2. jar包中没有将加解密相关的的class打进去
   
看样子需要进行二开了，唉。 clone项目到本地，拉个新分支。

首先看下源码，在`XBootEncryptor`中定义了springboot的classloader`final String jarLauncher = "org.springframework.boot.loader.JarLauncher";` 这里的jarlauncher在spring3中已经变包路径了。
没想到这么简单，心里暗喜。于是把这里修改为：`org.springframework.boot.loader.launch.JarLauncher`。 用spring3搭个demo, 重新打包。

再看jar文件， main-class已经写出去了，xjar相关的包也成功写到jar包。松了口气，看样子没太大问题。

开命令窗口，启动项目，一气呵成。看到终端输出springboot的logo时，心里已经松了口气。

可惜天不遂愿，又打印了几行日志后，抛出以下错误：
```
2024-09-27T17:51:36.403+08:00 ERROR 3796 --- [demo] [           main] o.s.boot.SpringApplication               : Application run failed

org.springframework.beans.factory.BeanDefinitionStoreException: Incompatible class format in URL [jar:nested:/D:/workspace/xiudianer/workspace/transport-mqtt-device-v4/branches/4.0.0/transport-mqtt-device-xjar/src/main/java/com/xd/device/ad/jars/encrypted5.jar/!BOOT-INF/classes/!/com/example/demo/DemoApplication.class]: set system property 'spring.classformat.ignore' to 'true' if you mean to ignore such files during classpath scanning
        at org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider.scanCandidateComponents(ClassPathScanningCandidateComponentProvider.java:504) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider.findCandidateComponents(ClassPathScanningCandidateComponentProvider.java:351) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.annotation.ClassPathBeanDefinitionScanner.doScan(ClassPathBeanDefinitionScanner.java:277) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.annotation.ComponentScanAnnotationParser.parse(ComponentScanAnnotationParser.java:128) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.annotation.ConfigurationClassParser.doProcessConfigurationClass(ConfigurationClassParser.java:306) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.annotation.ConfigurationClassParser.processConfigurationClass(ConfigurationClassParser.java:246) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.annotation.ConfigurationClassParser.parse(ConfigurationClassParser.java:197) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.annotation.ConfigurationClassParser.parse(ConfigurationClassParser.java:165) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.annotation.ConfigurationClassPostProcessor.processConfigBeanDefinitions(ConfigurationClassPostProcessor.java:417) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.annotation.ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry(ConfigurationClassPostProcessor.java:290) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors(PostProcessorRegistrationDelegate.java:349) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(PostProcessorRegistrationDelegate.java:118) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.support.AbstractApplicationContext.invokeBeanFactoryPostProcessors(AbstractApplicationContext.java:789) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:607) ~[spring-context-6.1.13.jar!/:6.1.13]
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:146) ~[spring-boot-3.3.4.jar!/:3.3.4]
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:754) ~[spring-boot-3.3.4.jar!/:3.3.4]
        at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:456) ~[spring-boot-3.3.4.jar!/:3.3.4]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:335) ~[spring-boot-3.3.4.jar!/:3.3.4]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1363) ~[spring-boot-3.3.4.jar!/:3.3.4]
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1352) ~[spring-boot-3.3.4.jar!/:3.3.4]
        at com.example.demo.DemoApplication.main(DemoApplication.java:11) ~[!/:0.0.1-SNAPSHOT]
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77) ~[na:na]
        at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
        at java.base/java.lang.reflect.Method.invoke(Method.java:568) ~[na:na]
        at org.springframework.boot.loader.launch.Launcher.launch(Launcher.java:91) ~[encrypted5.jar:0.0.1-SNAPSHOT]
        at org.springframework.boot.loader.launch.Launcher.launch(Launcher.java:53) ~[encrypted5.jar:0.0.1-SNAPSHOT]
        at io.xjar.boot.XJarLauncher.launch(XJarLauncher.java:27) ~[encrypted5.jar:0.0.1-SNAPSHOT]
        at io.xjar.boot.XJarLauncher.main(XJarLauncher.java:23) ~[encrypted5.jar:0.0.1-SNAPSHOT]
Caused by: org.springframework.core.type.classreading.ClassFormatException: ASM ClassReader failed to parse class file - probably due to a new Java class file version that is not supported yet. Consider compiling with a lower '-target' or upgrade your framework version. Affected class: URL [jar:nested:/D:/workspace/xiudianer/workspace/transport-mqtt-device-v4/branches/4.0.0/transport-mqtt-device-xjar/src/main/java/com/xd/device/ad/jars/encrypted5.jar/!BOOT-INF/classes/!/com/example/demo/DemoApplication.class]
        at org.springframework.core.type.classreading.SimpleMetadataReader.getClassReader(SimpleMetadataReader.java:59) ~[spring-core-6.1.13.jar!/:6.1.13]
        at org.springframework.core.type.classreading.SimpleMetadataReader.<init>(SimpleMetadataReader.java:48) ~[spring-core-6.1.13.jar!/:6.1.13]
        at org.springframework.core.type.classreading.SimpleMetadataReaderFactory.getMetadataReader(SimpleMetadataReaderFactory.java:103) ~[spring-core-6.1.13.jar!/:6.1.13]
        at org.springframework.core.type.classreading.CachingMetadataReaderFactory.getMetadataReader(CachingMetadataReaderFactory.java:122) ~[spring-core-6.1.13.jar!/:6.1.13]
        at org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider.scanCandidateComponents(ClassPathScanningCandidateComponentProvider.java:470) ~[spring-context-6.1.13.jar!/:6.1.13]
        ... 28 common frames omitted
Caused by: java.lang.IllegalArgumentException: null
        at org.springframework.asm.ClassReader.<init>(ClassReader.java:262) ~[spring-core-6.1.13.jar!/:6.1.13]
        at org.springframework.asm.ClassReader.<init>(ClassReader.java:180) ~[spring-core-6.1.13.jar!/:6.1.13]
        at org.springframework.asm.ClassReader.<init>(ClassReader.java:166) ~[spring-core-6.1.13.jar!/:6.1.13]
        at org.springframework.asm.ClassReader.<init>(ClassReader.java:287) ~[spring-core-6.1.13.jar!/:6.1.13]
        at org.springframework.core.type.classreading.SimpleMetadataReader.getClassReader(SimpleMetadataReader.java:56) ~[spring-core-6.1.13.jar!/:6.1.13]
        ... 32 common frames omitted

panic: exit status 1

```


一脸蒙蔽，这tm是什么错啊。

再看源码，xjar是自定义classloader来加载并解密类，核心就在`XBootClassLoader.findClass`。 这里源码中修改并打印下class的文件数据，看看究竟加载的什么鬼~

启动项目发现，class文件已解密，但为啥报错呢？只能使用万能断点大法了，在异常堆栈`SimpleMetadataReader.getClassReader`处断点，从这里入手。

该方法反编译源码如下：
```
    private static ClassReader getClassReader(Resource resource) throws IOException {
        InputStream is = resource.getInputStream();

        ClassReader var2;
        try {
            try {
                var2 = new ClassReader(is);
            } catch (IllegalArgumentException var5) {
                throw new ClassFormatException("ASM ClassReader failed to parse class file - probably due to a new Java class file version that is not supported yet. Consider compiling with a lower '-target' or upgrade your framework version. Affected class: " + resource, var5);
            }
        } catch (Throwable var6) {
            if (is != null) {
                try {
                    is.close();
                } catch (Throwable var4) {
                    var6.addSuppressed(var4);
                }
            }

            throw var6;
        }

        if (is != null) {
            is.close();
        }

        return var2;
    }

```
这里有看到inputstrem,于是断点把inputstream打印输出看下内容，发现这里读取的class文件居然是加密的。那么为啥这里没解密呢？ 继续往上跟堆栈。

根据参数Resource, 往上跟可以找到springboot扫描注解组件逻辑，也就是扫描项目中所有有注解的class，然后再调用这个方法读取加载class。 继续去看看这个Resource是怎么创建的。

一路跟，在`PathMatchingResourcePatternResolver.doFindPathMatchingJarResources`发现了创建`Resource`资源的代码`result.add(rootDirResource.createRelative(relativePath));`

```

    protected Set<Resource> doFindPathMatchingJarResources(Resource rootDirResource, URL rootDirUrl, String subPattern) throws IOException {
        URLConnection con = rootDirUrl.openConnection();
        JarFile jarFile;
        String jarFileUrl;
        String rootEntryPath;
        boolean closeJarFile;
        if (con instanceof JarURLConnection jarCon) {
            jarFile = jarCon.getJarFile();
            jarFileUrl = jarCon.getJarFileURL().toExternalForm();
            JarEntry jarEntry = jarCon.getJarEntry();
            rootEntryPath = jarEntry != null ? jarEntry.getName() : "";
            closeJarFile = !jarCon.getUseCaches();
        } else {
            String urlFile = rootDirUrl.getFile();

            try {
                int separatorIndex = urlFile.indexOf("*/");
                if (separatorIndex == -1) {
                    separatorIndex = urlFile.indexOf("!/");
                }

                if (separatorIndex != -1) {
                    jarFileUrl = urlFile.substring(0, separatorIndex);
                    rootEntryPath = urlFile.substring(separatorIndex + 2);
                    jarFile = this.getJarFile(jarFileUrl);
                } else {
                    jarFile = new JarFile(urlFile);
                    jarFileUrl = urlFile;
                    rootEntryPath = "";
                }

                closeJarFile = true;
            } catch (ZipException var17) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Skipping invalid jar class path entry [" + urlFile + "]");
                }

                return Collections.emptySet();
            }
        }

        try {
            if (logger.isTraceEnabled()) {
                logger.trace("Looking for matching resources in jar file [" + jarFileUrl + "]");
            }

            if (StringUtils.hasLength(rootEntryPath) && !rootEntryPath.endsWith("/")) {
                rootEntryPath = rootEntryPath + "/";
            }

            Set<Resource> result = new LinkedHashSet(64);
            Enumeration<JarEntry> entries = jarFile.entries();

            while(entries.hasMoreElements()) {
                JarEntry entry = (JarEntry)entries.nextElement();
                String entryPath = entry.getName();
                if (entryPath.startsWith(rootEntryPath)) {
                    String relativePath = entryPath.substring(rootEntryPath.length());
                    if (this.getPathMatcher().match(subPattern, relativePath)) {
                        result.add(rootDirResource.createRelative(relativePath)); // 注意这里，创建了resource资源
                    }
                }
            }

            LinkedHashSet var22 = result;
            return var22;
        } finally {
            if (closeJarFile) {
                jarFile.close();
            }

        }
    }
```


上面的代码创建了resource资源，于是继续跟进继续跟进`createRelative`方法，那么问题来了，为啥springboot2没问题，spring3读取出来的却又是加密的，这是为毛？

最简单的方法就是对比着看， 两个版本方法如下:

springboot3
```
    protected URL createRelativeURL(String relativePath) throws MalformedURLException {
        if (relativePath.startsWith("/")) {
            relativePath = relativePath.substring(1);
        }

        return ResourceUtils.toRelativeURL(this.url, relativePath);
    }
```

springboot2
```
	@Override
	public Resource createRelative(String relativePath) throws MalformedURLException {
		if (relativePath.startsWith("/")) {
			relativePath = relativePath.substring(1);
		}
		return new UrlResource(new URL(this.url, relativePath));
	}

```

上边代码一眼看，也没太大问题。都是通过url创建了一个资源链接而已，为毛就是跑步起来呢。 再次祭出断点大法。

断点后发现，两个创建的资源中, `URL` 属性中的`URLStreamHandler`有很大区别。springboot2中该属性为xjar的解密器，二springboot3中却是一个简单的文件读取器。 

为啥呢，喔翻了下源码发现在springboot2中，创建`Resouces`实例时，url属性实例是直接new出来的，当前类的解密器也就是`this.url`中`URLStreamHandler`，将会在`URL`的构造器中得到继承，所以新Resource读取时也就会解密了

但在springboot3中，`ResourceUtils.toRelativeURL`方法中创建URL时是先构建URI实例，再创建URL实例，这个过程中把`this.URLStreamHandler`丢失了。

原因找到了，看了下springboot github的issue，官方确实变更了，原因是因为springboot2中的`URL`的构造函数在之后的jdk20标记为废弃，所以就改了实现方法。不过官方表示下个版本会兼容处理这个问题，目前spring3最新版为`3.3.4`

但问题是我们现在就要用啊，只能拉个分支魔改了。

想想思路，既然旧版本没问题，那么喔先把`spring-core`包中的这个方法还原为老版本，看看spring能正常跑起来不。测试了下，没问题。

其实这里基本就没问题了，实际项目中只要把这个spring-core包从私仓中替换，项目加密也就没有太大的毛病，但是这样对于追求完美的我来说，方便性还差点。毕竟如果使用了springboot3的多个版本，不可能每个都去修改替换下啊，想想都好麻烦

那就再扩展下吧，既然自定义了classloader，那我可以在类加载过程中通过`asm`修改加载中的类，将`UrlResource.createRelative`方法替换为老版本。


由于asm是通过字节码来修改方法的，通过java源码来写出字节码，小弟还没有那个功力。

于是取个巧，直接使用`javap -verbose URLResource.class` 来查看老版本这个方法的指令流程
```
 public org.springframework.core.io.Resource createRelative(java.lang.String) throws java.net.MalformedURLException;
    descriptor: (Ljava/lang/String;)Lorg/springframework/core/io/Resource;
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=6, locals=2, args_size=2
         0: aload_1
         1: ldc           #36                 // String /
         3: invokevirtual #37                 // Method java/lang/String.startsWith:(Ljava/lang/String;)Z
         6: ifeq          15
         9: aload_1
        10: iconst_1
        11: invokevirtual #38                 // Method java/lang/String.substring:(I)Ljava/lang/String;
        14: astore_1
        15: new           #39                 // class org/springframework/core/io/UrlResource
        18: dup
        19: new           #13                 // class java/net/URL
        22: dup
        23: aload_0
        24: getfield      #6                  // Field url:Ljava/net/URL;
        27: aload_1
        28: invokespecial #40                 // Method java/net/URL."<init>":(Ljava/net/URL;Ljava/lang/String;)V
        31: invokespecial #41                 // Method "<init>":(Ljava/net/URL;)V
        34: areturn
      LineNumberTable:
        line 238: 0
        line 239: 9
        line 241: 15
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      35     0  this   Lorg/springframework/core/io/UrlResource;
            0      35     1 relativePath   Ljava/lang/String;
      StackMapTable: number_of_entries = 1
        frame_type = 15 /* same */
    Exceptions:
      throws java.net.MalformedURLException
```

然后使用asm转意下，以下为实现核心代码:
```


    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
        MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);

        if (name.equals("createRelative") && desc.equals("(Ljava/lang/String;)Lorg/springframework/core/io/Resource;")) {
            // 修改原方法的字节码
            mv = new MyMethodVisitor(mv) ;
        }

        return mv;
    }

    //MyMethodVisitor的结构
    public class MyMethodVisitor extends MethodVisitor {
        private final MethodVisitor target;

        public MyMethodVisitor(MethodVisitor mv) {
            super(ASM9, null);
            this.target = mv;
        }

        //此方法在目标方法调用之前调用，所以前置操作可以在这处理
        @Override
        public void visitCode() {
            target.visitCode();

            target.visitCode();
            target.visitVarInsn(ALOAD, 1);
            Label A = new Label();
            Label B = new Label();

            target.visitLdcInsn("/");
            target.visitMethodInsn(INVOKEVIRTUAL, "java/lang/String", "startsWith", "(Ljava/lang/String;)Z", false);
            target.visitJumpInsn(IFEQ, A);

            target.visitVarInsn(ALOAD, 1);
            target.visitInsn(ICONST_1);
            target.visitMethodInsn(INVOKEVIRTUAL, "java/lang/String", "substring", "(I)Ljava/lang/String;", false);
            target.visitVarInsn(ASTORE, 1);
            target.visitJumpInsn(GOTO,B);

            target.visitLabel(A);
            target.visitLabel(B);

            target.visitTypeInsn(Opcodes.NEW, "org/springframework/core/io/UrlResource");
            target.visitInsn(DUP);
            target.visitTypeInsn(Opcodes.NEW, "java/net/URL");
            target.visitInsn(DUP);
            target.visitVarInsn(ALOAD, 0);
            target.visitFieldInsn(GETFIELD, "org/springframework/core/io/UrlResource", "url", "Ljava/net/URL;");
            target.visitVarInsn(ALOAD, 1);
            target.visitMethodInsn(INVOKESPECIAL, "java/net/URL", "<init>", "(Ljava/net/URL;Ljava/lang/String;)V", false);
            target.visitMethodInsn(INVOKESPECIAL, "org/springframework/core/io/UrlResource", "<init>", "(Ljava/net/URL;)V", false);
            target.visitInsn(ARETURN); //

            target.visitMaxs(6, 2);
            target.visitEnd();


        }
        @Override
        public void visitMaxs(int maxStack, int maxLocals) {
            super.visitMaxs(maxStack + 1, maxLocals);
        }

    }
```

最后，由于classloader中使用了`asm`，所以也需要将asm提前打到加密jar包 中。 

然后再来编译一个加密包，再启动，成功！

这样道友们就可以直接使用项目，不用去修改`spring-core`包了。


项目传送地址：https://github.com/MisterChangRay/xjar4springboot3 
这个只支持springboot3哟~ 好用点个赞。把
