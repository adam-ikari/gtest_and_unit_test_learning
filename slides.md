---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: 编写可测试的代码
info: |
  ## 编写可测试代码指南
  基于mhevery的指南 - 四个使代码难以测试的设计缺陷

  Learn more at [Sli.dev](https://sli.dev)
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# open graph
seoMeta:
  # By default, Slidev will use ./og-image.png if it exists,
  # or generate one from the first slide if not found.
  ogImage: auto
  # ogImage: https://cover.sli.dev
---

# 编写可测试的代码

编写可测试代码指南

基于mhevery的指南 - 四个使代码难以测试的设计缺陷

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  按空格进入下一页 <carbon:arrow-right />
</div>

<div class="abs-br m-6 text-xl">
  <button @click="$slidev.nav.openInEditor()" title="在编辑器中打开" class="slidev-icon-btn">
    <carbon:edit />
  </button>
  <a href="https://github.com/slidevjs/slidev" target="_blank" class="slidev-icon-btn">
    <carbon:logo-github />
  </a>
</div>

<!--
每张幻灯片的最后一个注释块将被视为幻灯片备注。它将在演示者模式下与幻灯片一起可见和可编辑。[在文档中阅读更多](https://sli.dev/guide/syntax.html#notes)
-->

---

# 关于本指南

本指南确定了四个使代码难以测试的主要缺陷：

<v-clicks>

1. **构造函数执行实际工作**
2. **深入协作者**
3. **脆弱的全局状态和单例**
4. **类职责过多**

</v-clicks>

<br>
<v-click>

这些缺陷经常出现在现实世界的代码中，使单元测试变得困难或不可能。
理解这些模式有助于我们编写更易维护和可测试的代码。

</v-click>

---

# 缺陷 #1: 构造函数执行实际工作

## 问题所在

<v-clicks>

- 构造函数应该只将参数分配给字段
- 当构造函数执行实际工作时，会使得：
  - 在测试中创建实例变得困难
  - 用测试替身替换协作者变得困难
  - 理解类依赖关系变得困难

</v-clicks>

---

# 缺陷 #1: 构造函数执行实际工作 (继续)

## 警告信号

<v-clicks>

- 在构造函数中或字段声明中使用 `new` 关键字
- 在构造函数中或字段声明中调用静态方法
- 构造函数中除了字段赋值之外的任何操作
- 构造函数完成后对象未完全初始化
- 构造函数中存在控制流（条件或循环逻辑）
- 在构造函数中进行复杂的对象图构造
- 添加或使用初始化块

</v-clicks>

---

# 缺陷 #1: 示例

## 之前：难以测试

```cpp
class EmailSender {
private:
  SmtpClient client;  // 在构造函数中创建
  Logger logger;
  
public:
  EmailSender(string configPath) {
    // 构造函数中执行实际工作！
    Config config = new ConfigFileReader(configPath).read(); 
    this.client = new SmtpClient(config.getHost(), config.getPort());
    this.logger = new FileLogger("email.log");
    if (!client.isConnected()) {
      throw new ConnectionException();
    }
  }
};
```

<v-click>

问题：
- 无法模拟 SmtpClient 或 Logger
- 无法在没有网络连接的情况下进行测试
- 难以轻松测试错误处理路径

</v-click>

---

# 缺陷 #1: 示例

## 之后：可测试的设计

```cpp
class EmailSender {
private:
  SmtpClient& client;  // 注入的依赖
  Logger& logger;      // 注入的依赖
  
public:
  // 构造函数只分配字段
  EmailSender(SmtpClient& client, Logger& logger) 
    : client(client), logger(logger) {
  }
  
  void sendEmail(Email email) {
    logger.log("Sending email");
    client.send(email);
  }
};
```

<v-click>

优势：
- 易于在测试中注入模拟对象
- 构造函数中没有实际工作
- 依赖关系清晰

</v-click>

---

# 缺陷 #2: 深入协作者

## 问题所在

<v-clicks>

- 需要对象只是为了深入了解它们以获取其他对象的类
- 违反得墨忒耳定律
- 在类之间创建紧密耦合
- 使测试变得更加困难，因为您需要创建复杂的对象图

</v-clicks>

---

# 缺陷 #2: 深入协作者 (继续)

## 警告信号

<v-clicks>

- 传入的对象从未直接使用（仅用于获取其他对象）
- 违反得墨忒耳定律：方法调用链通过对象图走过不止一个点(.)
- 参数或字段中的可疑名称：
  - `context`
  - `environment`
  - `principal`
  - `container`
  - `manager`

</v-clicks>

---

# 缺陷 #2: 示例

## 之前：难以测试

```cpp
class UserRegistration {
private:
  DatabaseManager dbManager;
  
public:
  UserRegistration(DatabaseManager dbManager) 
    : dbManager(dbManager) {
  }
  
  void registerUser(UserData userData) {
    // 深入协作者
    Connection conn = dbManager.getConnectionPool().getConnection();
    UserRepository repo = new UserRepository(conn);
    repo.save(userData);
  }
};
```

<v-click>

问题：
- 需要模拟复杂的对象图 (DatabaseManager → ConnectionPool → Connection)
- 类之间紧密耦合
- 难以隔离测试

</v-click>

---

# 缺陷 #2: 示例

## 之后：可测试的设计

```cpp
class UserRegistration {
private:
  UserRepository& userRepository;
  
public:
  UserRegistration(UserRepository& userRepository) 
    : userRepository(userRepository) {
  }
  
  void registerUser(UserData userData) {
    // 直接使用协作者
    userRepository.save(userData);
  }
};
```

<v-click>

优势：
- 清晰的单一依赖
- 易于模拟 UserRepository
- 遵循得墨忒耳定律

</v-click>

---

# 缺陷 #3: 脆弱的全局状态和单例

## 问题所在

<v-clicks>

- 全局状态使代码不可预测
- 单例创建隐藏依赖
- 测试变得依赖顺序
- 难以并行运行测试
- 难以隔离被测系统

</v-clicks>

---

# 缺陷 #3: 脆弱的全局状态和单例 (继续)

## 警告信号

<v-clicks>

- 添加或使用单例
- 添加或使用静态字段或静态方法
- 添加或使用静态初始化块
- 添加或使用注册表
- 添加或使用服务定位器

</v-clicks>

---

# 缺陷 #3: 示例

## 之前：难以测试

```cpp
class OrderProcessor {
public:
  void processOrder(Order order) {
    // 使用全局状态和单例
    PaymentService.getInstance().charge(order.getAmount());
    InventoryManager.getInstance().updateStock(order.getItems());
    Logger.getLogger().log("Order processed: " + order.getId());
  }
};
```

<v-click>

问题：
- 无法模拟单例实例
- 测试通过全局状态相互影响
- 难以轻松测试错误场景
- 难以并行运行测试

</v-click>

---

# 缺陷 #3: 示例

## 之后：可测试的设计

```cpp
class OrderProcessor {
private:
  PaymentService& paymentService;
  InventoryManager& inventoryManager;
  Logger& logger;
  
public:
  OrderProcessor(PaymentService& paymentService,
                 InventoryManager& inventoryManager,
                 Logger& logger)
    : paymentService(paymentService),
      inventoryManager(inventoryManager),
      logger(logger) {
  }
  
  void processOrder(Order order) {
    paymentService.charge(order.getAmount());
    inventoryManager.updateStock(order.getItems());
    logger.log("Order processed: " + order.getId());
  }
};
```

<v-click>

优势：
- 依赖关系明确
- 易于注入模拟对象
- 无全局状态
- 测试可以独立运行

</v-click>

---

# 缺陷 #4: 类职责过多

## 问题所在

<v-clicks>

- 具有多个职责的类难以理解
- 难以测试所有场景
- 一个职责的更改可能破坏其他职责
- 违反单一职责原则

</v-clicks>

---

# 缺陷 #4: 类职责过多 (继续)

## 警告信号

<v-clicks>

- 总结类的作用时包含"和"字
- 新团队成员难以阅读并快速理解类的作用
- 类中的字段只在某些方法中使用
- 类中有只操作参数的静态方法

</v-clicks>

---

# 缺陷 #4: 示例

## 之前：难以测试

```cpp
class UserService {
private:
  DatabaseConnection conn;
  EmailService emailService;
  UserValidator validator;
  Cache cache;
  
public:
  User createUser(string name, string email) {
    // 验证逻辑
    if (!validator.isValidEmail(email)) {
      throw new ValidationException();
    }
    
    // 数据库逻辑
    User user = new User(name, email);
    conn.save(user);
    
    // 通知逻辑
    emailService.sendWelcomeEmail(user);
    
    return user;
  }
  
  // 其他用于不同职责的方法...
};
```

<v-click>

问题：
- 多个职责：验证、持久化、通知
- 难以隔离测试
- 每个测试需要复杂的设置

</v-click>

---

# 缺陷 #4: 示例

## 之后：可测试的设计

```cpp
// 拆分为专注的类
class UserValidator {
public:
  bool isValidEmail(string email) {
    // 只有验证逻辑
  }
};

class UserRepository {
private:
  DatabaseConnection& conn;
public:
  UserRepository(DatabaseConnection& conn) : conn(conn) {}
  
  void save(User user) {
    // 只有持久化逻辑
    conn.save(user);
  }
};

class WelcomeEmailSender {
private:
  EmailService& emailService;
public:
  WelcomeEmailSender(EmailService& emailService) 
    : emailService(emailService) {}
  
  void sendWelcomeEmail(User user) {
    // 只有通知逻辑
    emailService.sendWelcomeEmail(user);
  }
};
```

<v-click>

优势：
- 每个类都有单一职责
- 易于单独测试每个类
- 依赖关系清晰

</v-click>

---

# 测试的好处

当我们消除这些缺陷后，测试变得容易得多：

<v-clicks>

- **快速**：测试运行迅速
- **隔离**：测试之间互不影响
- **具体**：当测试失败时，我们知道确切的问题所在
- **清晰**：测试代码易于理解
- **可维护**：实现更改时测试不会中断

</v-clicks>

---

# 总结

<v-clicks>

1. **构造函数应该只将参数分配给字段**
   - 注入依赖而不是创建它们

2. **直接请求依赖**
   - 不要深入了解协作者以获取其他对象

3. **避免全局状态和单例**
   - 使依赖关系明确

4. **遵循单一职责原则**
   - 一个类，一个职责

</v-clicks>

<br>
<v-click>

> "防止bug最有效的方法是编写可测试的代码。"
> — Miško Hevery

</v-click>

---

# 参考资料

<v-clicks>

- Miško Hevery的[编写可测试代码指南](https://github.com/mhevery/guide-to-testable-code)
- [Google测试博客](https://testing.googleblog.com/)
- Robert C. Martin的《代码整洁之道》
- Kent Beck的《测试驱动开发》

</v-clicks>

<br>
<v-click>

<div class="text-center">

谢谢！

问答环节

</div>

</v-click>