# IResourceServiceProvider 考

Dec. 1, 2025

---

`IResourceServiceProvider` 通常是自定义语言服务不可或缺的一环，而鲜少有介绍该文件之内容，故撰此文。此文试图从 `IResourceServiceProvider` 出发，讨论 Xtext 中资源需要何种服务，以及为其所影响之模块

## 提供的服务

我们首先可以看下 `IResourceServiceProvider` 实现类 `DefaultResourceServiceProvider` 并观察一下这些资源服务提供商提供了哪些服务

```java
public class DefaultResourceServiceProvider implements IResourceServiceProvider, IResourceServiceProviderExtension {
	
	@Inject
	private IContainer.Manager containerManager;
	
	@Inject
	private IResourceDescription.Manager resourceDescriptionManager;
	
	@Inject
	private IResourceValidator resourceValidator;
	
	@Inject
	private FileExtensionProvider fileExtensionProvider;
	
	@Inject(optional = true)
	private IEncodingProvider encodingProvider;
	
	@Inject
	private Injector injector;
	
	@Override
	public org.eclipse.xtext.resource.IContainer.Manager getContainerManager() {
		return containerManager;
	}
	
	@Override
	public IResourceDescription.Manager getResourceDescriptionManager() {
		return resourceDescriptionManager;
	}
	
	@Override
	public IResourceValidator getResourceValidator() {
		return resourceValidator;
	}
	
	@Override
	public boolean canHandle(URI uri) {
		return fileExtensionProvider.isValid(uri.fileExtension());
	}

	@Override
	public IEncodingProvider getEncodingProvider() {
		return encodingProvider;
	}
	
	@Override
	public <T> T get(Class<T> t) {
		try {
			return injector.getInstance(t);
		} catch (ConfigurationException e) {
			return null;
		}
	}

	/**
	 * @since 2.9
	 */
	@Override
	public boolean isSource(URI uri) {
		return !uri.isArchive();
	}
	
}
```

可以看到，文件提供了 7 个方法。我们将从这些方法出发，探讨各自的作用

## 目录
1. [`getContainerManager`](#getcontainermanager)
2. [`getResourceDescriptionManager`](#getresourcedescriptionmanager)
    - [`getResourceDescription`](#getresourcedescription)
    - [`createDelta`](#createdelta)
    - [两个 `isAffected`](#两个-isaffected)
    - [`AllChangeAware`](#allchangeaware)
3. [`getResourceValidator`](#getresourcevalidator)
4. [`canHandle`](#canhandle)
5. [`getEncodingProvider`](#getencodingprovider)
6. [`get`](#get)
7. [`isSource`](#issource)

## `getContainerManager`

`IContainer` 是一个 Xtext 定义的概念，表示一个资源文件所处的容器/空间

```java
/**
 * A {@link IContainer container} describes resources that should be treated as visible
 * on the same level during the scoping stage. This depends on language implementations
 * in a way that a container that was obtained for a given resource may contain other resources
 * that would create other containers with distinct contents.
 * A container may be optimized by means of the {@link ISelectable}-contract. 
 */
```

可以看到，这个 `container` 实际上存储了同层级的所有资源（实际上是资源的描述 `IResourceDescription`）。这个层级并不是由地理位置上的层级决定，而是由语言本身的逻辑而决定：某些语言中，资源互相之间全局可见；某些语言中，定义文件互相之间全局可见，而代码文件只和对应的定义文件相互可见（这种情况下，对定义文件取 IContainer 和对其代码文件取 IContainer 明显内容不同）

而 `IContainer.Manager` 提供了根据资源找到其对应的 `container` 的能力，进而可以找到所有对当前资源暴露的其他资源的描述，进而缩减需要进行交叉引用的范围

## `getResourceDescriptionManager`

`IResourceDescriptionManager` 是生成 `IResourceDescription` 的管理类。因为 `Xtext` 使用懒加载逻辑以防止内存中存在过多资源（及其语法树等相关内容），生成描述的逻辑是重要的

实际上，这个管理类还承担了很多其他作用，我们来看接口代码

```java
@ImplementedBy(DefaultResourceDescriptionManager.class)
interface Manager {
    

    /**
     * @return a resource description for the given resource. The result represents the current state of the given
     *         resource.
     */
    IResourceDescription getResourceDescription(Resource resource);

    /**
     * @return a delta for both given descriptions.
     */
    IResourceDescription.Delta createDelta(IResourceDescription oldDescription, IResourceDescription newDescription);
    
    /**
     * @return whether the candidate is affected by the change in the delta.
     * @throws IllegalArgumentException
     *             if this manager is not responsible for the given candidate.
     */
    boolean isAffected(IResourceDescription.Delta delta, IResourceDescription candidate)
            throws IllegalArgumentException;

    /**
     * Batch operation to check whether a description is affected by any given delta in
     * the given context. Implementations may perform any optimizations to return <code>false</code> whenever
     * possible, e.g. check the deltas against the visible containers.
     * @param deltas List of deltas to check. May not be <code>null</code>.
     * @param candidate The description to check. May not be <code>null</code>.
     * @param context The current context of the batch operation. May not be <code>null</code>.
     * @return whether the candidate is affected by any of the given changes.
     * @throws IllegalArgumentException
     *             if this manager is not responsible for the given candidate.
     */
    boolean isAffected(Collection<IResourceDescription.Delta> deltas,
            IResourceDescription candidate,
            IResourceDescriptions context)
            throws IllegalArgumentException;

    /**
     * Implement this interface if your language should be notified of all {@link Delta}s, even
     * if they don't contain any changed {@link EObjectDescription}s
     * @since 2.7
     */
    interface AllChangeAware extends Manager {
        /**
         * Batch operation to check whether a description is affected by any given delta in
         * the given context. Implementations may perform any optimizations to return <code>false</code> whenever
         * possible, e.g. check the deltas against the visible containers.
         * @param deltas List of deltas to check. May not be <code>null</code>. In contrast to {@link #isAffected(Collection, IResourceDescription, IResourceDescriptions)}
         * callers of this method are expected to pass in all deltas, even if they don't have changed {@link IEObjectDescription}s
         * @param candidate The description to check. May not be <code>null</code>.
         * @param context The current context of the batch operation. May not be <code>null</code>.
         * @return whether the candidate is affected by any of the given changes.
         * @throws IllegalArgumentException
         *             if this manager is not responsible for the given candidate.
         */
        boolean isAffectedByAny(Collection<IResourceDescription.Delta> deltas,
                IResourceDescription candidate,
                IResourceDescriptions context)
                throws IllegalArgumentException;
    }
}
```

### `getResourceDescription`

生成资源描述

### `createDelta`

在构建过程中，如果当前资源处于脏状态（添加/更改/删除），则生成一个 $\Delta_{ResourceDescription}$ 来表示这次更新，方便后续进行增量

### 两个 `isAffected`

某些情况下，一个文件的更新会引起其他文件的更新。这种情况下，我们需要将增量的变化 $\Delta_{ResourceDescription}$ 列表与怀疑可能需要更新的资源进行碰撞，并确定这个被怀疑可能需要更新的资源需不需要进行一次构建

### `AllChangeAware`

我们可以观察一下 `Indexer` 的实现

```java
/**
 * Return true, if the given resource must be processed due to the given changes.
 */
protected boolean isAffected(IResourceDescription affectionCandidate, IResourceDescription.Manager manager,
        Collection<IResourceDescription.Delta> newDeltas, Collection<IResourceDescription.Delta> allDeltas,
        IResourceDescriptions resourceDescriptions) {
    if (manager instanceof IResourceDescription.Manager.AllChangeAware) {
        return ((IResourceDescription.Manager.AllChangeAware) manager).isAffectedByAny(allDeltas,
                affectionCandidate, resourceDescriptions);
    } else {
        if (newDeltas.isEmpty()) {
            return false;
        } else {
            return manager.isAffected(newDeltas, affectionCandidate, resourceDescriptions);
        }
    }
}
```

一般情况下，如果计算出来的 $\Delta_{ResourceDescription}$ 列表为空，即本次更新似乎没有更改任何文件描述的情况下，我们也会希望通知其他文件，这种情况下我们可以实现这个内部类，免于重写 `Indexer` 逻辑

## `getResourceValidator`

如何对当前资源进行校验——内容较多，另起一章

## `canHandle`

我们写代码的时候，一般区分不同语言是通过后缀名进行的。比如 `Main.java` 很明显使用的是 Java 语言，而 `RunApp.kt` 则使用的是 Kotlin 语言。如果一个语言服务器既要支持 Java 又要支持 Kotlin，则必须对两种不同的资源使用不同的逻辑。注入的 `FileExtensionProvider` 就是提供当前这个 `IResourceServiceProvider` 对应的资源的后缀名的类

而 `canHandle` 则是简单地调用这个 `FileExtensionProvider` 对后缀名进行校验，以确定当前资源是否可以被这个 `IResourceServiceProvider` 捕获并提供服务

```java
private Set<String> fileExtensions;

@Inject
protected void setExtensions(@Named(Constants.FILE_EXTENSIONS)String extensions) {
    String[] split = extensions.split(",");
    this.fileExtensions = Sets.newLinkedHashSet();
    for (String string : split) {
        this.fileExtensions.add(string);
    }
}
```

我们实际看一下 `FileExtensionProvider` 的代码，就会发现其最核心的逻辑在于 `setExtensions` 这个注入方法接受了一个被注解为 `Constants.FILE_EXTENSIONS` 的入参，并使用逗号分隔作为可接受的 `fileExtensions`。这个注解则是在对应的 `RuntimeIdeModule` 中进行注册的

```java
public void configureLanguageName(Binder binder) {
    binder.bind(String.class).annotatedWith(Names.named(Constants.LANGUAGE_NAME)).toInstance("org.example.jack.ecdl");
}

public void configureFileExtensions(Binder binder) {
    if (properties == null || properties.getProperty(Constants.FILE_EXTENSIONS) == null)
        binder.bind(String.class).annotatedWith(Names.named(Constants.FILE_EXTENSIONS)).toInstance("ecdl");
}
```

如果我们想让这个文件支持 `ecdl` 以外的其他语言，比如 `Circuit Description Language, cdl`，我们可以这样写

```java
public void configureFileExtensions(Binder binder) {
    if (properties == null || properties.getProperty(Constants.FILE_EXTENSIONS) == null)
        binder.bind(String.class).annotatedWith(Names.named(Constants.FILE_EXTENSIONS)).toInstance("ecdl,cdl");
}
```

## `getEncodingProvider`

`IEncodingProvider` 提供了资源文件使用的是什么编码，如果要支持 `🚗🌏 = ⭐` 这种编程语言，则需要设置为 `UTF-8` 之类的

## `get`

很明显是一个拿到其他注册类实例之入口，用户可以通过 `IResourceServiceProvider` 访问所有绑定信息，进而进行更多操作

## `isSource`

首先，我们先介绍一下 `uri.isArchive()` 是起什么作用

Xtext 底层使用了 `EMF` 建模，而 `EMF` 提供的 `URI` 中这个方法。这个方法是确认一个 `uri` 是否指向一个压缩包内的文件

```
archive:file:/home/user/lib/some.jar!/models/foo.xmi
```

比如上面的这个 `uri` 就是指向压缩包 `file:/home/user/lib/some.jar` 中的 `models/foo.xmi` 这个文件。仿照 Java 使用 jar 包的逻辑就很好理解——语言服务需要识别这个 jar 包里提供的类文件，但是这些类文件已经被编译，所以不属于源代码文件

这个方法主要是在构建过程中使用，以确定资源加载逻辑——是否应从压缩包中读取文件并建立资源，本文不做展开
