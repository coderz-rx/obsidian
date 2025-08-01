### 一、Bean生命周期

1、Spring Bean一般默认是单例模式。对于单例模式的Bean，它的生命周期从容器初始化后，到容器销毁，一直存在。

2、单例Bean（非懒加载）生命周期：容器启动 -> Bean创建（实例化） -> 依赖注入（注入属性或者引用的service依赖） -> 初始化（执行初始化方法，非必须） -> 准备就绪 -> 容器关闭则销毁。
![[Pasted image 20250619105726.png]]

3、非懒加载单例bean，在创建、依赖注入并执行初始化后，会将Bean存储到统一的缓存中，Map结构，key为bean的名称，这样在使用时，可以直接查询缓存获取Bean实例。
- 存储在 `DefaultSingletonBeanRegistry` 类的 `singletonObjects` 属性中（ConcurrentHashMap 类型）
- 结构：`Map<String, Object>`，Key 为 Bean 名称，Value 为实例对象
- 如果是声明的原型Bean，不进行缓存，每次 `getBean()` 时动态创建新实例

4、原型Bean使用时，要注意在使用结束后手动释放占用的资源，如数据库链接。因为Java 虚拟机在进行GC的时候只会回收堆内存的对象，释放内存空间，并不会处理相关的外部资源占用。

5、Spring容器中的单例Bean不会被GC回收。懒加载的单例Bean，在第一次调用getBean之后才会进行实例化等步骤。

6、单例Bean不保证线程安全，默认无状态设计，需自行保证线程安全。原型Bean天然线程隔离，是线程安全的。

### 二、Spring容器及IOC

1、Spring通过IoC容器管理Bean的生命周期，包括实例化、依赖注入、初始化到销毁的全过程。开发者无需手动创建对象，容器自动完成依赖关系的装配。
- 控制反转（IoC）是一种‌ 降低代码耦合度‌ 的设计原则，其核心思想是将对象创建和依赖管理的控制权从程序内部转移到外部容器。
- 传统开发中，对象主动创建其依赖（如 `new Service()`），而在 IoC 模式下，对象的创建和依赖注入由外部容器完成，程序只需声明依赖关系

2、‌IoC 主要通过 ‌依赖注入 实现，容器在运行时动态将依赖对象传递给目标组件（如通过构造器、Setter 或字段注入）。依赖注入方式‌
- ‌构造器注入：强制依赖，通过构造函数传递对象14。
- ‌Setter注入：可选依赖，通过setter方法赋值14。
- ‌字段注入‌：直接通过`@Autowired`注入字段，但不利于单元测试

3、‌解耦的实现路径‌
- 组件‌不直接依赖具体实现，而是依赖接口或抽象。
- 容器根据配置‌动态组装对象‌，替换依赖时无需修改业务代码。
- 对象无需关注依赖的创建细节，只需关注自身逻辑，提升模块独立性。

4、IOC如何工作的
- 容器初始化阶段
	- 加载配置元数据：容器读取XML、注解或Java配置类（如`@Configuration`），解析Bean的定义信息（如类路径、作用域、初始化方法等），生成`BeanDefinition`对象存储配置元数据
	- 创建BeanFactory：容器根据配置创建`BeanFactory`（基础容器）或`ApplicationContext`（高级容器，继承`BeanFactory`），后者支持国际化、事件发布等扩展功能
- Bean生命周期管理（参照上面内容）
- 容器工作流程（以 ApplicationContext 为例）：启动容器 -> 加载配置元数据 -> 解析生成BeanDefinition -> 创建BeanFactory -> 调用BeanFactoryPostProcessor扩展 -> 实例化Bean -> 依赖注入 -> 执行BeanPostProcessor前置处理 -> 初始化Bean ->  执行BeanPostProcessor后置处理（AOP代理生成） -> Bean就绪 -> 容器关闭时销毁Bean

5、SpringBoot和Spring的区别：
- Spring Boot 与 Spring 的区别远不止于用注解替代 XML 配置，其本质是‌对 Spring 的深度封装和扩展‌，旨在简化开发流程、提升效率。（举个最明显的例子，Spring mvc中需要做的DispatcherServlet，在boot中会根据约定大于配置的原则进行默认配置）
- Spring可以根据项目依赖（如 `spring-boot-starter-data-jpa`）‌**自动装配 Bean 和组件**‌，无需手动定义数据源、事务管理器等。例如引入 H2 依赖后自动配置内存数据库。（==但也可以自定义配置对默认配置进行覆盖==）
- Spring Boot 不是替代 Spring‌，而是基于 Spring 的‌开发加速器‌：
	- 保留 Spring 核心（IoC、AOP、事务）。
	- 通过默认约定和自动装配‌降低使用门槛。
	- 专注于业务逻辑而非基础设施配置 。