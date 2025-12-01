# 自定义 ecore 文件并侧载 EMF 模型

Dec 1, 2025

---

## 背景

在使用 [`Xtext`](https://eclipse.dev/Xtext/) + [`LSP4J`](https://projects.eclipse.org/projects/technology.lsp4j) 进行[语言服务器](https://microsoft.github.io/language-server-protocol/)开发的时候，框架会为 DSL 中的每一个元素生成一个 `EObject` 类型的接口以及其对应实现类。这其实是因为 LSP4J 框架实际依赖了 [`EMF`](https://projects.eclipse.org/projects/modeling.emf.emf) 框架，并通过框架动态加载 `Resource` 解析后生成这些 `EObject` 实例。

但是，在实际开发过程中，语言服务可能需要扩展并非属于 DSL 的语言。考虑以下几种情况：

> 我需要一个 JSON/XML 格式的配置文件，读取其中的配置项以便于提示这些功能 \
> 我有一个图形化编程语言，并非 `Xtext` 支持的纯文本编程语言 \
> 我需要创建一个虚拟资源，参与构建流程（或生成共享描述 `exportedObjects` 等）

这种情况下，很难或根本无法写出另一个 `.xtext` 语法文件对应新增的资源，此时最简单的方法是侧载一个 `EMF` 模型以最大程度上利用框架，减少代码复杂程度。

本文将依托于一个以 `JSON` 表示的图形化编程语言为例，建立一个相应的 `EMF` 模型，侧载到 `Xtext` 已有的模型中，并且进行一个简要的类型导出（生成 `exportedObjects`），并进行类似于 `Xtext` 中使用 `@Check` 进行的语义校验。本文将基于 `Java` 而非 `Xtend` 语言。

## `Eclipse GLSP`

> 这种情况下，为何不使用 Eclipse 基金会的 [`GLSP`](https://eclipse.dev/glsp/) 呢？

因为我们预设的场景是——Jack 有一个已存在的 LSP 服务器。Eclipse GLSP 虽然提供了较为强大的图形化编程语言的语言服务适配，也有一个支持了 GLSP 的编辑器，但是这问题很大。

在 `Xtext` 中，编辑器充当读写磁盘的角色——语言服务器在这种情况下是不写文件内容的。这保证了即使语言服务器与编辑器断连，用户也可以修改代码

GLSP Server 控制图模型的状态，因此图的编辑操作需要 server 在线才能生效。因此断连后只存在 Client 的情况下，无法继续编辑

在 `LSP` 中，语言服务器是二等公民，而 `GLSP` 中却翻身做主人。但是我们通常希望编辑器可以独立于语言服务器存在，因此我们不选择使用 `GLSP`，而是自己实现一个类似 `GLSP` 的侧载方法。

## 前置

读者需要可以使用 `Eclipse IDE for Java and DSL Developers` 或在 `Eclipse` 中安装 `EMF` 相关插件。强烈推荐使用已经提供好所有需要的相关工具的 `Eclipse IDE for Java and DSL Developers` 进行相关的操作。

## 场景

`Jack` 正在创建一个电子元件描述语言 `ECDL (Electic Component Description Language)`。电子元件通常有一定的输入引脚、一定的输出引脚，并且大多内部有构成电子元件的小元件。一个常见的电子元件建模可以由以下 Java 代码表示：

```Java
public class Component {
    String name;
    String id;
    List<Pin> inputs;
    List<Pin> outputs;
    List<Component> children;
}

public class Pin {
    String name;
    String id;
}
```

我们将帮助 `Jack` 创建一个分析 `ECDL` 的语言服务器组成部分。

## 目录

1. [使用 Eclipse 创建 ecore 文件](#使用-Eclipse-创建-ecore-文件)
2. [依据 `ecore` 文件生成 `EObject` 代码](#依据-ecore-文件生成-eobject-代码)
3. [侧载 `edcl`](#侧载-ecdl)
4. [将前端模型转换为 Ecore 模型并加载至 `EcdlResource` 资源文件中](#将前端模型转换为-ecore-模型并加载至-ecdlresource-资源文件中)
5. （可选）[计算属性并注册 `eAdapter` 到 `eObject` 中](#计算属性并注册-eadapter-到-eobject-中)
6. [导出一个成员描述](#导出一个成员描述)
7. [为 `ecdl` 写一个简单的验证器](#为-ecdl-写一个简单的验证器)

## 使用 Eclipse 创建 ecore 文件

新建 `Ecore Model` 文件，展开并编辑 `package` 的属性

```
Name: ecdl
Ns Prefix: ecdl
Ns URI: http://www.example.org/jack/ecdl
```

右键元素并选择 `EClass`，添加 `Component` 和 `Pin` 两个类。

选择 `Component` 类，右键选择 `EAttribute`，依次添加 `name`, `id`。至于 EType 字段，我们需要选择对应的类型，这里是字符串类型的变量，所以选择 `EString`，以 `name` 为例，其属性字段需要设置如下

```
EType: EString
Name: name
```

选择 `Component` 类，右键选择 `EReference`，依次添加 `inputs`, `outputs` 和 `children` 三个子成员。选择 `EReference` 的原因是这些都是自定义类型，而非基本数据类型，需要引用其他类。以 `inputs` 为例，其属性字段需要设置如下

```
Containment: true       // 这决定着后续生成的 EObject 是否会被关联在对应的 Resource 下
EType: Pin
Lower Bound: 0          // 最少可以有 0 个输入引脚
Upper Bound: -1         // -1 代表着无上限，-2 代表着未定义
```

同理，建立 `Pin` 类。

## 依据 `ecore` 文件生成 `EObject` 代码

新建 `EMF Generator Model` 文件，选择上一步生成的 `ecore` 模型文件并确认。可能需要设置的地方有

```
// 最外层元素（对应 genmodel 文件属性）
Model:
    Model Directory: ...    // 生成模型的根目录

// 内部的 `ecdl` 包（对应 ecore 文件属性）
All:
    Base Package: ...       // 生成模型的包
```

右键最外层元素并点击 `Generate Model Code` 以生成 Java 代码

## 侧载 `ecdl`

在 `org.example.lsp.ide` 中建立新的包 `org.example.lsp.ecdl` 并生成模型代码后，我们需要生成对应的资源文件以读取文件内容并转换为内存模型。我们需要为这个包建立一个 `EcdlStandaloneSetup` 文件并注册到 `/META-INF/services/org.eclipse.xtext.ISetup` 文件中。在语言服务器启动时，其会检索该文件并将所有包加载进来（见 `LanguageServerImpl#initialize(params)` 方法）

```java
package org.example.lsp.ecdl;

public class EcdlStandaloneSetup implements ISetup {
    @Override
    public Injector createInjectorAndDoEMFRegistration() {
        Injector injector = Guice.createInjector(new EcdlRuntimeModule());

        IResourceFactory resourceFactory = injector.getInstance(IResourceFactory.class);
        IResourceServiceProvider serviceProvider = injector.getInstance(IResourceServiceProvider.class);

        Resource.Factory.Registry.INSTANCE.getExtensionToFactoryMap().put("ecdl", resourceFactory);
        IResourceServiceProvider.Registry.INSTANCE.getExtensionToFactoryMap().put("ecdl", serviceProvider);

        return injector;
    }
}
```

```java
package org.example.lsp.ecdl;

public class EcdlRuntimeModule extends DefaultRuntimeModule {

    public void configureLanguageName(Binder binder) {
        binder.bind(String.class)
                .annotatedWith(Names.named(Constants.LANGUAGE_NAME))
                .toInstance("org.example.jack.ecdl");
    }

    public void configureFileExtensions(Binder binder) {
        binder.bind(String.class)
                .annotatedWith(Names.named(Constants.FILE_EXTENSIONS))
                .toInstance("ecdl");
    }

    public Class<? extends IResourceFactory> bindIResourceFactory() {
        return EcdlResourceFactory.class;
    }

    public Class<? extends IResourceServiceProvider> bindIResourceServiceProvider() {
        return EcdlResourceServiceProvider.class;
    }

    public Class<? extends IResourceDescription.Manager> bindIResourceDescriptionManager() { 
        return EcdlResourceDescriptionManager.class;
    }

    public Class<? extends IDefaultResourceDescriptionStrategy> bindResourceDescriptionStrategy() { 
        return EcdlResourceDescriptionStrategy.class;
    }
}
```

`EcdlRuntimeModule` 中是需要绑定的信息，其中为了完成侧载需要额外创建的类的大致为上述的文件。但是，由于这些类中大部分需要注入其他的依赖项，所以我们可能还需要注入其他的实现，如 `QualifiedNameProvider` 等。完整的注入信息可以参考 `Xtext` 自己生成的 `RuntimeModule`

|名称|作用|
|---|---|
|EcdlResourceFactory|用于自定义 `Resource` 应如何创建|
|EcdlResourceServiceProvider|自定义如何对 `Resource` 提供校验等服务|
|EcdlResourceDescriptionManager|`Resource` 可能会导出成员描述，以供其他文件使用（如全局变量、函数等）|
|EcdlResourceDescriptionStrategy|为 `Resource` 中生成的 `EObject` 创建描述、建立引用等|

但一切的基础是一个资源文件 `EcdlResource`，我们需要建立一个对应的 `XtextResource` 的子类以保证每次前端模型更改时，对应的资源文件会被正确地加载出来（无论是通过 `didFileChange` 事件同步，还是在资源首次从硬盘加载）


## 将前端模型转换为 Ecore 模型并加载至 `EcdlResource` 资源文件中

在资源文件中，我们可以通过重写 `doLoad` 方法达成自定义模型生成之目的。我们假设 `Ecdl` 前端图形化实现是以 JSON 格式存储的，那么我们可以通过 `Gson` 或者 `Jackson` 等反序列化工具生成 `POJO` 模型。在生成已有模型的 `EObject` 后需要将其放入对应资源文件的 `contents` 中，这样每个 `EObject` 访问 `eResource` 时都会对应该资源文件

```java
public class EcdlResource extends XtextResource {

    @Inject
    NodeModelBuilder nodeModelBuilder;
    @Inject
    Gson gson;

    @Override
    protected void doLoad(InputStream inputStream, Map<?, ?> options) throws IOException {
        // 在此方法中写入自定义加载方法
        String text = new BufferedReader(new InputStreamReader(inputStream, StandardCharsets.UTF_8))
            .lines().collect(Collectors.joining("\n"));
        ICompositeNode node = this.nodeModelBuilder.newRootNode(text);
        // 生成 POJO 模型（也可以使用其他方式实现，后文会谈及）
        Model model = this.gson.fromJson(text, Model.class);
        // 转换 POJO 模型为 ecore 模型
        // 将 POJO 模型转换为 ecore 模型的方式依赖于反射可以很轻松地使用 AI 生成或者使用现有工具，不做赘述
        Converter converter = new Converter(EcdlPackage.eINSTANCE);
        EObject rootEObject = converter.convert(model);
        // 将 ecore 模型与当前资源文件关联起来
        this.setParseResult(new ParseResult(rootEObject, node, false));
        this.getContents().clear();
        this.getContents().add(rootEObject);
    }
}
```

## 计算属性并注册 `eAdapter` 到 `eObject` 中

假设每一个 `ecdl` 元素都有对应的唯一 ID 可以用于跳转，那么在转换模型时，计算该 ID 并添加到 `eObject` 中的 `eAdapters` 将是比较标准的做法

```java
public class EcdlIDAdapter extends AdapterImpl {

    private final String id;

    public EcdlIDAdapter(String id) {
        this.id = id;
    }

    public String getId() {
        return id;
    }
}
```

这种途径适合计算如 ID 这样的轻量级的全局唯一不变的属性，不适合添加实际业务（因为每次文件发生变动都会重新计算，不能保证增量，耗费巨大的情况下不建议使用）

## 导出一个成员描述

在这个部分，笔者会简单地完成一个对 `Resource` 建立 `ResourceDescription` 并对其中的 `eObject` 建立 `EObjectDescription` 的例子，以备参考

首先，需要确定的是，建立 `ResourceDescription` 的逻辑在对应的 `IResourceDescription.Manager` 实例中。我们在 `EcdlRuntimeModule` 中绑定的实现类 `EcdlResourceDescriptionManager` 就是生成其之入口

如果需要重写这部分逻辑，则需要如此实现这个接口

```java
public class EcdlResourceDescriptionManager implements IResourceDescription.Manager {
    @Override
	public IResourceDescription getResourceDescription(final Resource resource) {
		// 可以在这里自定义生成方法
	}
}
```

而我们一般可以使用 `Xtext` 自带的 `DefaultResourceDescriptionManager`

```java
private static final String CACHE_KEY = DefaultResourceDescriptionManager.class.getName() + "#getResourceDescription";

@Override
public IResourceDescription getResourceDescription(final Resource resource) {
    return cache.get(CACHE_KEY, resource, new Provider<IResourceDescription>() {
        @Override
        public IResourceDescription get() {
            return internalGetResourceDescription(resource, strategy);
        }
    });
}

protected IResourceDescription internalGetResourceDescription(Resource resource, IDefaultResourceDescriptionStrategy strategy) {
    return new DefaultResourceDescription(resource, strategy, cache);
}
```

生成的 `IResourceDescription` 是一个对当前资源的整体性描述。`Xtext` 实行的是懒加载模式，资源在未被影响的情况下都会以描述的形式存在，而不会建立语法树。我们可以重写 `DefaultResourceDescription` 来自定义你需要对文件本身进行什么样的描述

在对文件生成描述后，我们有时希望对语法树的部分成员进行描述——这些成员可能定义了一个类型、变量等。我们需要让这些成员在即使对应的资源文件不存在内存中（即处于懒加载状态）时也可以被其他资源访问，故而我们需要导出一些成员的描述 `EObjectDescription`

我们首先看下 `DefaultResourceDescription` 是如何导出 `EObjectDescription` 的（内容有删减）

```java
public class DefaultResourceDescription extends AbstractResourceDescription {

	private IDefaultResourceDescriptionStrategy strategy;

	@Override
	protected List<IEObjectDescription> computeExportedObjects() {
		if (!getResource().isLoaded()) {
			try {
				getResource().load(null);
			} catch (IOException e) {
				log.error(e.getMessage(), e);
				return Collections.<IEObjectDescription> emptyList();
			}
		}
		final List<IEObjectDescription> exportedEObjects = newArrayList();
		IAcceptor<IEObjectDescription> acceptor = new IAcceptor<IEObjectDescription>() {
			@Override
			public void accept(IEObjectDescription eObjectDescription) {
				exportedEObjects.add(eObjectDescription);
			}
		};
		TreeIterator<EObject> allProperContents = EcoreUtil.getAllProperContents(getResource(), false);
		while (allProperContents.hasNext()) {
			EObject content = allProperContents.next();
            // 这里使用了一个在初始化时传入的 strategy 全局变量来进行构建
			if (!strategy.createEObjectDescriptions(content, acceptor))
				allProperContents.prune();
		}
		return exportedEObjects;
	}
}
```

可以看到，这里使用的是 `IDefaultResourceDescriptionStrategy` 来为资源文件中的所有 `EObject` 生成描述信息。我们不妨看一下其默认实现 `DefaultResourceDescriptionStrategy`

```java
@Singleton
public class DefaultResourceDescriptionStrategy implements IDefaultResourceDescriptionStrategy {

	@Inject
	private IQualifiedNameProvider qualifiedNameProvider;

	@Override
	public boolean createEObjectDescriptions(EObject eObject, IAcceptor<IEObjectDescription> acceptor) {
		if (getQualifiedNameProvider() == null)
			return false;
		try {
			QualifiedName qualifiedName = getQualifiedNameProvider().getFullyQualifiedName(eObject);
			if (qualifiedName != null) {
				acceptor.accept(EObjectDescription.create(qualifiedName, eObject));
			}
		} catch (Exception exc) {
			LOG.error(exc.getMessage(), exc);
		}
		return true;
	}
}
```

这里我们可以看出，其为所有 `EObject` 生成了一个 `QualifiedName`，而我们只需要重写该方法即可

```java
@Singleton
public class EcdlResourceDescriptionStrategy implements IDefaultResourceDescriptionStrategy {

	@Override
	public boolean createEObjectDescriptions(EObject eObject, IAcceptor<IEObjectDescription> acceptor) {
		if (eObject instanceOf Component) {
            calculateComponentEObjectDescription((Component) eObject, acceptor);
        }
        // 注意这里如果返回 false 则会立刻打断 DefaultResourceDescription 中的遍历
        // 如果使用默认的 DefaultResourceDescription 则需要谨慎对待返回值含义
        return true;
	}
}
```

## 为 `ecdl` 写一个简单的验证器

语言服务器本身一个很重要的功能就是对资源进行验证：是否存在语法错误，是否存在语义错误，是否存在可以提升的点。在 `Xtext` 原生资源中进行校验时，你可以在任何继承了 `AbstractDeclarativeValidator` 的类中进行如下的操作

```java
@Check
public void check(EObject eObject) {
    this.error(eObject, "我心情不好的时候真的会对所有的 eObject 都报错", null);
}
```

但是侧载的 `ecdl` 并不能被 `Xtext` 框架识别并认为侧载的 `EPackage` 是其的一部分，继承该类并不能够实现对整个项目中所有某类型的 `eObject` 进行校验。一定要通过这种方法校验复杂性过大（读者不妨阅读一下 `IResourceServiceProvider` 以及其实现类中涉及 `IResourceValidator` 的源码），取而代之的是，我们自定义对资源进行校验的方法

在继承了 `IResourceServiceProvider` 的 `EcdlResourceServiceProvider` 中，重写 `getResourceValidator` 方法

```java
@Inject
public EcdlResourceServiceProvider(Set<EcdlValidator> validators) {
    this.validators = validators;
}

IResourceValidator iResourceValidator = (resource, mode, indicator) -> {
    List<Issue> issues = new LinkedList<>();
    for (EcdlValidator validator : validators) {
        issues.addAll(validator.validate((EcdlResource) resource));
    }
    return issues;
};

@Override
public IResourceValidator getResourceValidator() {
    return iResourceValidator;
}
```

同时，因为初始化函数需要接受所有自定义的 `EcdlValidator`，需要在 `EcdlRuntimeModule` 中配置这些类

```java
public void configureValidators(Binder binder) {
    Multibinder<EcdlValidator> multibinder = Multibinder.newSetBinder(binder, EcdlValidator.class);
    multibinder.addBinding().to(MyValidator.class);
    multibinder.addBinding().to(YAEV.class);    // Yet Another Ecdl Validator
}
```

这样就可以为 Ecdl 资源添加任意多数量的验证器了，读者也可以尝试通过反射、定义 Annotation 以达到类似于 Xtext 的 `@Check` 风格的校验，此处不再赘述