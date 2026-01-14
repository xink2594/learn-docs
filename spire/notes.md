# spire
## 依赖注入
### 基本概念
``` c++
import spire_extensions_injection.*

main(): Int64 {
    // 1. 定义服务描述集合
    let services = ServiceCollection()
    // 2. 注册服务
    services.addSingleton<DbContext, DbContext>()
    services.addSingleton<IDbConnection, MySqlConnection>()
    // 3. 构建容器
    let provider = services.build()
    // 4. 解析服务
    let connection = provider.getOrThrow<DbContext>()
    return 0
}

public interface IDbConnection  { }

public class MySqlConnection <: IDbConnection{ }

public class DbContext {
    // 由容器注入
    public DbContext(let connection: IDbConnection) {
    }
}
```
发生了什么？

- 向容器索要 DbContext

- 容器查看 DbContext 的定义，发现它的构造函数需要一个 IDbConnection

- 容器自动去它的服务列表里查阅，是否有注册过 IDbConnection 

- 发现你注册过 services.addSingleton<IDbConnection, MySqlConnection>()

- 于是容器先创建了 MySqlConnection 的实例，然后自动把它塞进 DbContext 的构造函数里。

- 交付 DbContext

这个过程就是 **依赖注入** ，DbContext 依赖 IDbConnection，容器把这个依赖注入进去了。

### 服务注册
#### 基础注册
``` c++
// 完整描述符方式
services.add(ServiceDescriptor.singleton<IDbConnection, MySqlConnection>())
//详细描述：注册一个名为IDbConnection的服务，需要有MysqlConnection....

// 简化泛型方式（推荐）
services.addSingleton<IDbConnection, MySqlConnection>()

// 类型信息方式（适合动态注册）
services.addSingleton(TypeInfo.of<IDbConnection>(), TypeInfo.of<MySqlConnection>())

// 现有实例
services.addSingleton<IDbConnection>(MySqlConnection())

// 工厂模式（推荐性能最佳，无需反射参与）
services.addSingleton<DbContext>{ ServiceProvider => 
    DbContext(ServiceProvider.getOrThrow<IDbConnection>())
}
```

#### 协调器

适用场景：复杂的注册需求

假设我们有一个服务A,它依赖了B,C,D,E服务，同时依赖了一个字符串。 已知容器已经注册了B,C,D,E服务，如果之间注册服务A，就会报错。那么如何注册服务A？

答：我们可以使用ActivatorUtilities，它是容器协调器，支持提供未注册的服务一起完成服务解析。

``` c++
services.addSingleton(B())
services.addSingleton(C())
services.addSingleton(D())
services.addSingleton(E())

services.addSingleton<A>{sp => 
    // 容器中未注册的服务，通过第二个参数提供
    return ActivatorUtilities.CreateInstance<A>(sp, "spire")
}

public class A {
    public A(name: String, a: B, c: C, d: D, e: E) {

    }
}
```

### 服务解析

依赖注入框架提供多种服务解析方式，并使用不同的内部机制来创建和管理服务实例。

#### 解析必需服务

确保服务已注册时使用：

```c++
let services = ServiceCollection()
services.addSingleton<IDbConnection, MysqlConnection>()
let provider = services.build()

// 如果服务不存在，将抛出异常
let connection = provider.getOrThrow<IDbConnection>()
```

#### 解析可选服务

不确定服务是否注册时使用：

```c++
let services = ServiceCollection()
let provider = services.build()

// 如果服务解析失败返回None
let connection = provider.get<IDbConnection>()
```

#### 解析多实现服务

```c++
interface IDbConnection {}
class MysqlConnection <: IDbConnection {}
class SqlConnection <: IDbConnection {}

let services = ServiceCollection()

// 注册多个数据库链接实现
services.addSingleton<IDbConnection, SqlConnection>()
services.addSingleton<IDbConnection, MysqlConnection>()

let provider = services.build()

// 获取所有注册的链接实现
let connections = provider.getAll<IDbConnection>()
```

#### 解析构造器依赖

```c++
interface IDbConnection {}
class MysqlConnection <: IDbConnection {}
class SqlConnection <: IDbConnection {}

class DbContext {
    public DbContext(let connections: Collection<IDbConnection>) {

    }
}

let services = ServiceCollection()

// 注册多个数据库链接实现
services.addSingleton<IDbConnection, SqlConnection>()
services.addSingleton<IDbConnection, MysqlConnection>()
services.addSingleton<DbContext, DbContext>()
let provider = services.build()

// 获取所有注册的链接实现
let context = provider.getOrThrow<DbContext>()
```

#### 协调器

解析未注册但依赖容器的服务：

```c++
interface IDbConnection {}
class MysqlConnection <: IDbConnection {}
class SqlConnection <: IDbConnection {}

class DbContext {
    public DbContext(tenant: String, let connections: Collection<IDbConnection>) {

    }
}

let services = ServiceCollection()

// 注册多个数据库链接实现
services.addSingleton<IDbConnection, SqlConnection>()
services.addSingleton<IDbConnection, MysqlConnection>()
// services.addSingleton<DbContext, DbContext>() 不注册
let provider = services.build()

// 获取所有注册的链接实现，并可以提供额外参数
let context = ActivatorUtilities.createInstance<DbContext>(provider, "spire")
```

> 注意：由于DbContext并未注册到容器，而是通过ActivatorUtilities创建的，DbContext实例是不受容器托管的。即失去了生命周期管理。

### 生命周期

对于基于构造器解析的服务满足如下约束

1. 单例服务不能依赖非单例服务
2. 不能从根容器解析非单例服务

| 生命周期  | 作用范围             | 典型应用场景       |
| :-------- | :------------------- | :----------------- |
| Singleton | 整个应用程序生命周期 | 配置服务、日志服务 |
| Scoped    | 单个作用域范围内     | 数据库上下文       |
| Transient | 每次请求创建新实例   | 轻量级临时服务     |

#### 作用域示例



```c++
import std.random.*
import spire_extensions_injection.*

main(): Int64 {
    let services = ServiceCollection()
    services.addSingleton<Singleton, Singleton>()
    services.addScoped<Scoped, Scoped>()
    services.addTransient<Transient, Transient>()
    let provider = services.build()
    // 创建作用域1
    println("===============Scope2====================")
    try (scope1 = provider.createScope()) {
        let singleton = scope1.services.getOrThrow<Singleton>()
        let scoped = scope1.services.getOrThrow<Scoped>()
        let transient = scope1.services.getOrThrow<Transient>()
        println(scope1.services.getOrThrow<Singleton>().id)
        println(scope1.services.getOrThrow<Singleton>().id)
        println(scope1.services.getOrThrow<Scoped>().id)
        println(scope1.services.getOrThrow<Scoped>().id)
        println(scope1.services.getOrThrow<Transient>().id)
        println(scope1.services.getOrThrow<Transient>().id)
    }
    println("===============Scope2====================")
    // 创建作用域2
    try (scope2 = provider.createScope()) {
        let singleton = scope2.services.getOrThrow<Singleton>()
        let scoped = scope2.services.getOrThrow<Scoped>()
        let transient = scope2.services.getOrThrow<Transient>()
        println(scope2.services.getOrThrow<Singleton>().id)
        println(scope2.services.getOrThrow<Singleton>().id)
        println(scope2.services.getOrThrow<Scoped>().id)
        println(scope2.services.getOrThrow<Scoped>().id)
        println(scope2.services.getOrThrow<Transient>().id)
        println(scope2.services.getOrThrow<Transient>().id)
    }
    return 0
}

public class Singleton {
    public var id: String = "Singleton:" + Random().nextInt64().toString()
}

public class Scoped {
    public var id: String = "Scoped:" + Random().nextInt64().toString()
}

public class Transient {
    public var id: String = "Transient:" +  Random().nextInt64().toString()
}
```

```
===============Scope2====================
Singleton:-6446799371252424971
Singleton:-6446799371252424971
Scoped:-8239741083227323237
Scoped:-8239741083227323237
Transient:-455999103032063464
Transient:-3732090699255222433
===============Scope2====================
Singleton:-6446799371252424971
Singleton:-6446799371252424971
Scoped:2953664531395606013
Scoped:2953664531395606013
Transient:3501588865321937698
Transient:7833323014559001106
```

#### 资源释放

运行下面的示例，可以发现当作用域结束的时候，容器会自动释放服务，如果实现了Resource那么会调用它的close方法。



```
main(): Int64 {
    let services = ServiceCollection()
    services.addScoped<IDbConnection, SqlConnection>()
    let provider = services.build()
    // 创建作用域
    try (scope = provider.createScope()){
        scope.services.getOrThrow<IDbConnection>()
    }
    // 作用域结束
    return 0
}

interface IDbConnection {}

class MysqlConnection <: IDbConnection {}

class SqlConnection <: IDbConnection & Resource {
    private var _isClosed = false
    public func close() {
        println("链接已关闭")
    }
    public func isClosed() {
        _isClosed
    }
}
```
