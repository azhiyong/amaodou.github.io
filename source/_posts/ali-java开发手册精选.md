---
title: ali Java开发手册精选
date: 2020-05-09 11:09:13
tags: Java
---

## 编程规约

### 命名风格

1. 【强制】POJO 类中布尔类型的变量，都不要加 is 前缀，否则部分框架解析会引起序列化错误。

   反例：定义为基本数据类型 Boolean isDeleted 的属性，它的方法也是 isDeleted()，RPC
   框架在反向解析的时候，“误以为”对应的属性名称是 deleted，导致属性获取不到，进而抛
   出异常。

   <!--more-->

2. 【参考】各层命名规约：

   A) Service/DAO 层方法命名规约

   1. 获取单个对象的方法用 get 做前缀。
   2. 获取多个对象的方法用 list 做前缀，复数形式结尾如：listObjects。
   3. 获取统计值的方法用 count 做前缀。
   4. 插入的方法用 save/insert 做前缀。
   5. 删除的方法用 remove/delete 做前缀。
   6. 修改的方法用 update 做前缀。

   B) 领域模型命名规约

   1. 数据对象：xxxDO，xxx 即为数据表名。
   2. 数据传输对象：xxxDTO，xxx 为业务领域相关的名称。
   3. 展示对象：xxxVO，xxx 一般为网页名称。
   4. POJO 是 DO/DTO/BO/VO 的统称，禁止命名成 xxxPOJO。

### OOP 规约

1. 【强制】Object 的 equals 方法容易抛空指针异常，应使用常量或确定有值的对象来调用
   equals。

   正例："test".equals(object);

   反例：object.equals("test");

   说明：推荐使用 java.util.Objects#equals（JDK7 引入的工具类）

2. 关于基本数据类型与包装数据类型的使用标准如下：

   1. 【强制】所有的 POJO 类属性必须使用包装数据类型。
   2. 【强制】RPC 方法的返回值和参数必须使用包装数据类型。
   3. 【推荐】所有的局部变量使用基本数据类型。

   说明：POJO 类属性没有初值是提醒使用者在需要使用时，必须自己显式地进行赋值，任何
   NPE 问题，或者入库检查，都由使用者来保证。

   正例：数据库的查询结果可能是 null，因为自动拆箱，用基本数据类型接收有 NPE 风险。

   反例：比如显示成交总额涨跌情况，即正负 x%，x 为基本数据类型，调用的 RPC 服务，调用
   不成功时，返回的是默认值，页面显示为 0%，这是不合理的，应该显示成中划线。所以包装
   数据类型的 null 值，能够表示额外的信息，如：远程调用失败，异常退出。

3. 【强制】定义 DO/DTO/VO 等 POJO 类时，不要设定任何属性默认值。

   反例：POJO 类的 gmtCreate 默认值为 new Date()，但是这个属性在数据提取时并没有置入具
   体值，在更新其它字段时又附带更新了此字段，导致创建时间被修改成当前时间。

4. 【强制】序列化类新增属性时，请不要修改 serialVersionUID 字段，避免反序列失败；如
   果完全不兼容升级，避免反序列化混乱，那么请修改 serialVersionUID 值。

   说明：注意 serialVersionUID 不一致会抛出序列化运行时异常。

5. 【推荐】循环体内，字符串的连接方式，使用 StringBuilder 的 append 方法进行扩展。

   说明：下例中，反编译出的字节码文件显示每次循环都会 new 出一个 StringBuilder 对象，
   然后进行 append 操作，最后通过 toString 方法返回 String 对象，造成内存资源浪费。

   反例：

   ```java
   String str = "start";
   for (int i = 0; i < 100; i++) {
       str = str + "hello";
   }
   ```

### 集合处理

1. 【强制】关于 hashCode 和 equals 的处理，遵循如下规则：

   1. 只要重写 equals，就必须重写 hashCode。
   2. 因为 Set 存储的是不重复的对象，依据 hashCode 和 equals 进行判断，所以 Set 存储的
      对象必须重写这两个方法。
   3. 如果自定义对象作为 Map 的键，那么必须重写 hashCode 和 equals。

   说明：String 重写了 hashCode 和 equals 方法，所以我们可以非常愉快地使用 String 对象
   作为 key 来使用。

2. 【强制】ArrayList 的 subList 结果不可强转成 ArrayList，否则会抛出 ClassCastException
   异常，即 java.util.RandomAccessSubList cannot be cast to java.util.ArrayList。

   说明：subList 返回的是 ArrayList 的内部类 SubList，并不是 ArrayList 而是 ArrayList
   的一个视图，对于 SubList 子列表的所有操作最终会反映到原列表上。

3. 【强制】在 subList 场景中，高度注意对原集合元素的增加或删除，均会导致子列表的遍历、
   增加、删除产生 ConcurrentModificationException 异常。

4. 【强制】使用集合转数组的方法，必须使用集合的 toArray(T[] array)，传入的是类型完全
   一样的数组，大小就是 list.size()。

   说明：使用 toArray 带参方法，入参分配的数组空间不够大时，toArray 方法内部将重新分配
   内存空间，并返回新数组地址；如果数组元素个数大于实际所需，下标为[ list.size() ]
   的数组元素将被置为 null，其它数组元素保持原值，因此最好将方法入参数组大小定义与集
   合元素个数一致。

   正例：

   ```java
   List<String> list = new ArrayList<String>(2);
   list.add("guan");
   list.add("bao");
   String[] array = new String[list.size()];
   array = list.toArray(array);
   ```

   反例：直接使用 toArray 无参方法存在问题，此方法返回值只能是 Object[]类，若强转其它
   类型数组将出现 ClassCastException 错误。

5. 【强制】使用工具类 Arrays.asList()把数组转换成集合时，不能使用其修改集合相关的方
   法，它的 add/remove/clear 方法会抛出 UnsupportedOperationException 异常。

   说明：asList 的返回对象是一个 Arrays 内部类，并没有实现集合的修改方法。Arrays.asList
   体现的是适配器模式，只是转换接口，后台的数据仍是数组。

   ```java
   String[] str = new String[] { "you", "wu" };
   List list = Arrays.asList(str);
   ```

   第一种情况：list.add("yangguanbao"); 运行时异常。

   第二种情况：str[0] = "gujin"; 那么 list.get(0)也会随之修改。

6. 【强制】泛型通配符<? extends T>来接收返回的数据，此写法的泛型集合不能使用 add 方
   法，而<? super T>不能使用 get 方法，作为接口调用赋值时易出错。

   说明：扩展说一下 PECS(Producer Extends Consumer Super)原则：第一、频繁往外读取内
   容的，适合用<? extends T>。第二、经常往里插入的，适合用<? super T>。

7. 【强制】不要在 foreach 循环里进行元素的 remove/add 操作。remove 元素请使用 Iterator
   方式，如果并发操作，需要对 Iterator 对象加锁。

   正例：

   ```java
   List<String> list = new ArrayList<>();
   list.add("1");
   list.add("2");
   Iterator<String> iterator = list.iterator();
   while (iterator.hasNext()) {
       String item = iterator.next();
       if (删除元素的条件) {
           iterator.remove();
       }
   }
   ```

   反例：

   ```java
   for (String item : list) {
       if ("1".equals(item)) {
           list.remove(item);
       }
   }
   ```

   说明：以上代码的执行结果肯定会出乎大家的意料，那么试一下把“1”换成“2”，会是同样的
   结果吗？

8. 【强制】在 JDK7 版本及以上，Comparator 实现类要满足如下三个条件，不然 Arrays.sort，
   Collections.sort 会报 IllegalArgumentException 异常。

   说明：三个条件如下

   1. x，y 的比较结果和 y，x 的比较结果相反。
   2. x>y，y>z，则 x>z。
   3. x=y，则 x，z 比较结果和 y，z 比较结果相同。

   反例：下例中没有处理相等的情况，实际使用中可能会出现异常：

   ```java
   new Comparator<Student>() {
       @Override
       public int compare(Student o1, Student o2) {
       return o1.getId() > o2.getId() ? 1 : -1;
       }
   };
   ```

9. 【推荐】集合初始化时，指定集合初始值大小。

   说明：HashMap 使用 HashMap(int initialCapacity) 初始化。

   正例：initialCapacity = (需要存储的元素个数 / 负载因子) + 1。注意负载因子（即 loader
   factor）默认为 0.75，如果暂时无法确定初始值大小，请设置为 16（即默认值）。

   反例：HashMap 需要放置 1024 个元素，由于没有设置容量初始大小，随着元素不断增加，容
   量 7 次被迫扩大，resize 需要重建 hash 表，严重影响性能。

10. 【推荐】高度注意 Map 类集合 K/V 能不能存储 null 值的情况，如下表格：

    | 集合类            | Key           | Value         | Super       | 说明                   |
    | ----------------- | ------------- | ------------- | ----------- | ---------------------- |
    | Hashtable         | 不允许为 null | 不允许为 null | Dictionary  | 线程安全               |
    | ConcurrentHashMap | 不允许为 null | 不允许为 null | AbstractMap | 锁分段技术（JDK8:CAS） |
    | TreeMap           | 不允许为 null | 允许为 null   | AbstractMap | 线程不安全             |
    | HashMap           | 允许为 null   | 允许为 null   | AbstractMap | 线程不安全             |

    反例： 由于 HashMap 的干扰，很多人认为 ConcurrentHashMap 是可以置入 null 值，而事实上，
    存储 null 值时会抛出 NPE 异常。

### 并发处理

1. 【强制】线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样
   的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。

   说明：Executors 返回的线程池对象的弊端如下：

   1. FixedThreadPool 和 SingleThreadPool:
      允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM。
   2. CachedThreadPool 和 ScheduledThreadPool:
      允许的创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致 OOM。

2. 【强制】SimpleDateFormat 是线程不安全的类，一般不要定义为 static 变量，如果定义为
   static，必须加锁，或者使用 DateUtils 工具类。

   正例：注意线程安全，使用 DateUtils。亦推荐如下处理：

   ```java
   private static final ThreadLocal<DateFormat> df = new ThreadLocal<DateFormat>() {
       @Override
       protected DateFormat initialValue() {
           return new SimpleDateFormat("yyyy-MM-dd");
       }
   };
   ```

   说明：如果是 JDK8 的应用，可以使用 Instant 代替 Date，LocalDateTime 代替 Calendar，
   DateTimeFormatter 代替 SimpleDateFormat，官方给出的解释：simple beautiful strong
   immutable thread-safe。

3. 【推荐】使用 CountDownLatch 进行异步转同步操作，每个线程退出前必须调用 countDown
   方法，线程执行代码注意 catch 异常，确保 countDown 方法被执行到，避免主线程无法执行
   至 await 方法，直到超时才返回结果。

   说明：注意，子线程抛出异常堆栈，不能在主线程 try-catch 到。

4. 【参考】 HashMap 在容量不够进行 resize 时由于高并发可能出现死链，导致 CPU 飙升，在
   开发过程中可以使用其它数据结构或加锁来规避此风险。

### 注释规约

1. 【强制】类、类属性、类方法的注释必须使用 Javadoc 规范，使用/\*_内容_/格式，不得使用
   // xxx 方式。

   说明：在 IDE 编辑窗口中，Javadoc 方式会提示相关注释，生成 Javadoc 可以正确输出相应注
   释；在 IDE 中，工程调用方法时，不进入方法即可悬浮提示方法、参数、返回值的意义，提高
   阅读效率。

2. 【参考】特殊注释标记，请注明标记人与标记时间。注意及时处理这些标记，通过标记扫描，
   经常清理此类标记。线上故障有时候就是来源于这些标记处的代码。

   1. 待办事宜（TODO）:（ 标记人，标记时间，[预计处理时间]）
      表示需要实现，但目前还未实现的功能。这实际上是一个 Javadoc 的标签，目前的 Javadoc
      还没有实现，但已经被广泛使用。只能应用于类，接口和方法（因为它是一个 Javadoc 标签）。
   2. 错误，不能工作（FIXME）:（标记人，标记时间，[预计处理时间]）
      在注释中用 FIXME 标记某代码是错误的，而且不能工作，需要及时纠正的情况。

## 异常日志

1. 【强制】异常不要用来做流程控制，条件控制。

   说明：异常设计的初衷是解决程序运行中的各种意外情况，且异常的处理效率比条件判断方式
   要低很多。

2. 【强制】不要在 finally 块中使用 return。

   说明：finally 块中的 return 返回后方法结束执行，不会再执行 try 块中的 return 语句。

3. 【推荐】方法的返回值可以为 null，不强制返回空集合，或者空对象等，必须添加注释充分
   说明什么情况下会返回 null 值。

   说明：本手册明确防止 NPE 是调用者的责任。即使被调用方法返回空集合或者空对象，对调用
   者来说，也并非高枕无忧，必须考虑到远程调用失败、序列化失败、运行时异常等场景返回
   null 的情况。

## 单元测试

1. 【强制】好的单元测试必须遵守 AIR 原则。

   说明：单元测试在线上运行时，感觉像空气（AIR）一样并不存在，但在测试质量的保障上，
   却是非常关键的。好的单元测试宏观上来说，具有自动化、独立性、可重复执行的特点。

   - A：Automatic（自动化）
   - I：Independent（独立性）
   - R：Repeatable（可重复）

2. 【强制】单元测试是可以重复执行的，不能受到外界环境的影响。

   说明：单元测试通常会被放到持续集成中，每次有代码 check in 时单元测试都会被执行。如
   果单测对外部环境（网络、服务、中间件等）有依赖，容易导致持续集成机制的不可用。

   正例：为了不受外界环境影响，要求设计代码时就把 SUT 的依赖改成注入，在测试时用 spring
   这样的 DI 框架注入一个本地（内存）实现或者 Mock 实现。

3. 【推荐】编写单元测试代码遵守 BCDE 原则，以保证被测试模块的交付质量。

   - B：Border，边界值测试，包括循环边界、特殊取值、特殊时间点、数据顺序等。
   - C：Correct，正确的输入，并得到预期的结果。
   - D：Design，与设计文档相结合，来编写单元测试。
   - E：Error，强制错误信息输入（如：非法数据、异常流程、非业务允许输入等），并得
     到预期的结果。

## MySQL 数据库

### 索引规约

1. 【强制】业务上具有唯一特性的字段，即使是多个字段的组合，也必须建成唯一索引。

   说明：不要以为唯一索引影响了 insert 速度，这个速度损耗可以忽略，但提高查找速度是明
   显的；另外，即使在应用层做了非常完善的校验控制，只要没有唯一索引，根据墨菲定律，必
   然有脏数据产生。

2. 【强制】在 varchar 字段上建立索引时，必须指定索引长度，没必要对全字段建立索引，根据
   实际文本区分度决定索引长度即可。

   说明：索引的长度与区分度是一对矛盾体，一般对字符串类型数据，长度为 20 的索引，区分
   度会高达 90%以上，可以使用 count(distinct left(列名, 索引长度))/count(\*)的区分度
   来确定。

3. 【强制】页面搜索严禁左模糊或者全模糊，如果需要请走搜索引擎来解决。

   说明：索引文件具有 B-Tree 的最左前缀匹配特性，如果左边的值未确定，那么无法使用此索
   引。

4. 【推荐】利用延迟关联或者子查询优化超多分页场景。

   说明：MySQL 并不是跳过 offset 行，而是取 offset+N 行，然后返回放弃前 offset 行，返回
   N 行，那当 offset 特别大的时候，效率就非常的低下，要么控制返回的总页数，要么对超过
   特定阈值的页数进行 SQL 改写。

   正例：先快速定位需要获取的 id 段，然后再关联：

   ```sql
   SELECT a.* FROM 表 1 a, (select id from 表 1 where 条件 LIMIT 100000,20 ) b where a.id=b.id
   ```

5. 【推荐】SQL 性能优化的目标：至少要达到 range 级别，要求是 ref 级别，如果可以是 consts
   最好。

   说明：

   1. consts 单表中最多只有一个匹配行（主键或者唯一索引），在优化阶段即可读取到数据。
   2. ref 指的是使用普通的索引（normal index）。
   3. range 对索引进行范围检索。

   反例：explain 表的结果，type=index，索引物理文件全扫描，速度非常慢，这个 index 级
   别比较 range 还低，与全表扫描是小巫见大巫。

### SQL 语句

1. 【强制】不要使用 count(列名)或 count(常量)来替代 count(_)，count(_)是 SQL92 定义的
   标准统计行数的语法，跟数据库无关，跟 NULL 和非 NULL 无关。

   说明：count(\*)会统计值为 NULL 的行，而 count(列名)不会统计此列为 NULL 值的行。

2. 【强制】count(distinct col) 计算该列除 NULL 之外的不重复行数，注意 count(distinct
   col1, col2) 如果其中一列全为 NULL，那么即使另一列有不同的值，也返回为 0。

3. 【强制】当某一列的值全是 NULL 时，count(col)的返回结果为 0，但 sum(col)的返回结果为
   NULL，因此使用 sum()时需注意 NPE 问题。

   正例：可以使用如下方式来避免 sum 的 NPE 问题：

   ```sql
   SELECT IF(ISNULL(SUM(g)),0,SUM(g)) FROM table;
   ```

4. 【强制】使用 ISNULL()来判断是否为 NULL 值。

   说明：NULL 与任何值的直接比较都为 NULL。

   1. NULL<>NULL 的返回结果是 NULL，而不是 false。
   2. NULL=NULL 的返回结果是 NULL，而不是 true。
   3. NULL<>1 的返回结果是 NULL，而不是 true。

5. 【推荐】in 操作能避免则避免，若实在避免不了，需要仔细评估 in 后边的集合元素数量，控
   制在 1000 个之内。

6. 【参考】如果有国际化需要，所有的字符存储与表示，均以 utf-8 编码，注意字符统计函数
   的区别。

   说明：

   SELECT LENGTH("轻松工作")； 返回为 12

   SELECT CHARACTER_LENGTH("轻松工作")； 返回为 4

   如果需要存储表情，那么选择 utf8mb4 来进行存储，注意它与 utf-8 编码的区别。

### ORM 映射

1. 【强制】POJO 类的布尔属性不能加 is，而数据库字段必须加 is\_，要求在 resultMap 中进行
   字段与属性之间的映射。

   说明：参见定义 POJO 类以及数据库字段定义规定，在`<resultMap>`中增加映射，是必须的。
   在 MyBatis Generator 生成的代码中，需要进行对应的修改。

2. 【推荐】不要写一个大而全的数据更新接口。传入为 POJO 类，不管是不是自己的目标更新字
   段，都进行 `update table set c1=value1,c2=value2,c3=value3;` 这是不对的。执行 SQL
   时，不要更新无改动的字段，一是易出错；二是效率低；三是增加 binlog 存储。

## 工程结构

### 应用分层

1. 【参考】分层领域模型规约：

   - DO（Data Object）：此对象与数据库表结构一一对应，通过 DAO 层向上传输数据源对象。
   - DTO（Data Transfer Object）：数据传输对象，Service 或 Manager 向外传输的对象。
   - BO（Business Object）：业务对象，由 Service 层输出的封装业务逻辑的对象。
   - AO（Application Object）：应用对象，在 Web 层与 Service 层之间抽象的复用对象模型，
     极为贴近展示层，复用度不高。
   - VO（View Object）：显示层对象，通常是 Web 向模板渲染引擎层传输的对象。
   - Query：数据查询对象，各层接收上层的查询请求。注意超过 2 个参数的查询封装，禁止
     使用 Map 类来传输。

### 二方库依赖

1. 【强制】二方库版本号命名方式：主版本号.次版本号.修订号

   1. 主版本号：产品方向改变，或者大规模 API 不兼容，或者架构不兼容升级。
   2. 次版本号：保持相对兼容性，增加主要功能特性，影响范围极小的 API 不兼容修改。
   3. 修订号：保持完全兼容性，修复 BUG、新增次要功能特性等。

   说明：注意起始版本号必须为：1.0.0，而不是 0.0.1 正式发布的类库必须先去中央仓库进
   行查证，使版本号有延续性，正式版本号不允许覆盖升级。如当前版本：1.3.3，那么下一个
   合理的版本号：1.3.4 或 1.4.0 或 2.0.0

2. 【强制】二方库里可以定义枚举类型，参数可以使用枚举类型，但是接口返回值不允许使用枚
   举类型或者包含枚举类型的 POJO 对象。

3. 【推荐】所有 pom 文件中的依赖声明放在`<dependencies>`语句块中，所有版本仲裁放在
   `<dependencyManagement>`语句块中。

   说明：`<dependencyManagement>`里只是声明版本，并不实现引入，因此子项目需要显式的声
   明依赖，version 和 scope 都读取自父 pom。而`<dependencies>`所有声明在主 pom 的
   `<dependencies>`里的依赖都会自动引入，并默认被所有的子项目继承。

### 服务器

1. 【推荐】高并发服务器建议调小 TCP 协议的 time_wait 超时时间。

   说明：操作系统默认 240 秒后，才会关闭处于 time_wait 状态的连接，在高并发访问下，服
   务器端会因为处于 time_wait 的连接数太多，可能无法建立新的连接，所以需要在服务器上
   调小此等待值。

   正例：在 linux 服务器上请通过变更/etc/sysctl.conf 文件去修改该缺省值（秒）：
   net.ipv4.tcp_fin_timeout = 30

## 设计规约

1. 【推荐】谨慎使用继承的方式来进行扩展，优先使用聚合/组合的方式来实现。

   说明：不得已使用继承的话，必须符合里氏代换原则，此原则说父类能够出现的地方子类一定
   能够出现，比如，“把钱交出来”，钱的子类美元、欧元、人民币等都可以出现。

[1]: https://github.com/alibaba/p3c
