---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: 编写可测试的代码
info: |
  四个使代码难以测试的设计缺陷

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

四个使代码难以测试的设计缺陷

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  按空格进入下一页 <carbon:arrow-right />
</div>

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

# 重要概念解释

在深入学习这四个缺陷之前，我们需要先理解一些重要概念：

<v-clicks>

- **协作者(Collaborator)**：类为了完成其职责而需要与其他类进行交互，这些交互的类被称为协作者。

- **依赖注入(Dependency Injection)**：通过构造函数或方法参数将依赖传递给类，而不是在类内部创建依赖。这使代码更加灵活和可测试，因为我们可以在测试时传入模拟对象。

- **模拟对象(Mock)**：在测试中用来替代真实对象的特殊对象，可以验证方法调用和设置期望行为。模拟对象帮助我们隔离被测试的代码单元，使测试更加可靠。

- **得墨忒耳定律(Law of Demeter)**：一个对象应该只与直接朋友交谈，不与陌生人的陌生人交谈，避免链式调用。

</v-clicks>

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

## 警告信号及说明

- **在构造函数中或字段声明中使用 `new` 关键字**

  ```cpp
  class UserService {
    // 在字段声明中使用new
    private: UserRepository* repo = new DatabaseUserRepository();
  };
  ```

  说明：这种方式将创建依赖的职责放在了类内部，使得难以替换为测试替身，也隐藏了类的真实依赖。

---

# 缺陷 #1: 构造函数执行实际工作 (继续)

## 警告信号及说明 (继续)

- **在构造函数中或字段声明中调用静态方法**

  ```cpp
  class OrderService {
    private: Logger logger;
    public: OrderService() : logger(LoggerFactory::getLogger()) {}
  };
  ```

  说明：静态方法调用创建了隐式依赖，同样难以替换为测试替身，且使类与特定实现紧密耦合。

---

# 缺陷 #1: 构造函数执行实际工作 (继续)

## 警告信号及说明 (继续)

- **构造函数中除了字段赋值之外的任何操作**

  ```cpp
  class EmailService {
    public: EmailService(string configPath) {
      // 除了字段赋值还有其他操作
      ConfigReader reader(configPath);
      this->config = reader.read();  // 复杂操作
      this->initSmtp();              // 方法调用
    }
  };
  ```

  说明：构造函数应该只负责初始化字段，而不是执行复杂的业务逻辑或调用其他方法。

---

# 缺陷 #1: 构造函数执行实际工作 (继续)

## 警告信号及说明 (继续)

- **构造函数完成后对象未完全初始化**

  ```cpp
  class PaymentProcessor {
    public: PaymentProcessor() {
      // 构造函数为空，需要额外调用init()
    }
    public: void init() { /* 初始化逻辑 */ }
  };
  ```

  说明：对象应该在构造函数完成后就处于完全可用状态，不需要额外的初始化步骤。

---

# 缺陷 #1: 构造函数执行实际工作 (继续)

## 警告信号及说明 (继续)

- **构造函数中存在控制流（条件或循环逻辑）**

  ```cpp
  class ReportGenerator {
    public: ReportGenerator(ReportType type) {
      // 构造函数中有条件逻辑
      if (type == ReportType::PDF) {
        this->formatter = new PdfFormatter();
      } else if (type == ReportType::CSV) {
        this->formatter = new CsvFormatter();
      }
    }
  };
  ```

  说明：构造函数中包含条件逻辑使类的行为变得复杂且难以预测，也增加了测试的复杂性。

---

# 缺陷 #1: 构造函数执行实际工作 (继续)

## 警告信号及说明 (继续)

- **在构造函数中进行复杂的对象图构造**

  ```cpp
  class ShoppingCart {
    public: ShoppingCart(User user) {
      // 复杂的对象图构造
      this->user = user;
      this->discountService = new DiscountService(
        new UserDiscountProvider(user),
        new SeasonalDiscountProvider(),
        new CouponDiscountProvider()
      );
    }
  };
  ```

  说明：在构造函数中构造复杂的对象图使得类承担了过多职责，也使测试需要创建大量依赖对象。

---

# 缺陷 #1: 构造函数执行实际工作 (继续)

## 警告信号及说明 (继续)

- **添加或使用初始化块**

  ```cpp
  class DataProcessor {
    private: vector<string> filters;

    // 初始化块
    public: DataProcessor() {
      filters.push_back("filter1");
      filters.push_back("filter2");
      filters.push_back("filter3");
    }
  };
  ```

  说明：初始化块中的逻辑应该移到专门的方法中，保持构造函数的简洁。

---

# 缺陷 #1: 示例

## 之前：难以测试

<div class="grid cols-2 gap-4">
<div>

```cpp
class EmailSender {
private:
  SmtpClient client;  // 在构造函数中创建
  Logger logger;

public:
  EmailSender(string configPath) {
    // 构造函数中执行实际工作！
    Config config =
      new ConfigFileReader(configPath).read();
    this.client = new SmtpClient(
        config.getHost(),
        config.getPort());
    this.logger = new FileLogger("email.log");
    if (!client.isConnected()) {
      throw new ConnectionException();
    }
  }
};
```

</div>
<div>

问题：

- 无法模拟 "SmtpClient" 或 "Logger"
- 无法在没有网络连接的情况下进行测试
- 难以轻松测试错误处理路径

</div>
</div>

---

# 缺陷 #1: 示例

## 之后：可测试的设计

<div class="grid cols-2 gap-4">
<div>

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

</div>
<div>

优势：

- 易于在测试中注入模拟对象
- 构造函数中没有实际工作
- 依赖关系清晰

</div>
</div>

---

# 缺陷 #2: 深入协作者

## 问题所在

::v-clicks

- 类需要其他对象只是为了获取更多的对象（深入协作者）
- 违反得墨忒耳定律
- 创建类之间的紧密耦合
- 使测试变得更加困难，因为您需要创建复杂的对象图

::

---

# 缺陷 #2: 深入协作者 (继续)

## 警告信号及说明

- **传入的对象从未直接使用（仅用于获取其他对象）**

  ```cpp
  class OrderService {
    public: void processOrder(Order order, DatabaseManager dbManager) {
      // dbManager仅用于获取Connection
      Connection conn = dbManager.getConnection();
      OrderRepository repo = new OrderRepository(conn);
      repo.save(order);
    }
  };
  ```

  说明：传入 "DatabaseManager" 只是为了获取 "Connection" ，这表明类与 "DatabaseManager" 的耦合度过高，应该直接依赖所需的对象。

---

- **违反得墨忒耳定律：方法调用链通过对象图走过不止一个点(.)**

  ```cpp
  class UserService {
    public: void sendNotification(User user) {
      // 违反得墨忒耳定律的链式调用
      user.getProfile().getPreferences().getNotificationSettings().getEmail();
    }
  };
  ```

  说明：这种链式调用称为"火车残骸"，增加了代码的脆弱性，任何一个环节的改变都可能影响整个调用链。

---

- **参数或字段中的可疑名称**

  ```cpp
  class ReportGenerator {
    private: ApplicationContext context;  // 可疑的"上下文"名称

    public: void generateReport() {
      // 深入协作者
      ReportConfig config = context.getConfiguration().getReportSettings();
    }
  };
  ```

  说明：像 "context"、"environment"、"manager" 这样的名称通常表明类可能在深入协作者，应该明确需要的具体依赖。

---

# 缺陷 #2: 示例对比

### 之前：难以测试

<div class="grid cols-2 gap-4">
<div>

```cpp
class UserRegistration {
private:
  DatabaseManager dbManager;

public:
  UserRegistration(DatabaseManager dbManager)
    : dbManager(dbManager) {
  }

  void registerUser(UserData userData) {
    // 深入协作者：
    // 通过dbManager获取ConnectionPool，再获取Connection
    Connection conn = dbManager
      .getConnectionPool()
      .getConnection();
    UserRepository repo = new UserRepository(conn);
    repo.save(userData);
  }
};
```

</div>
<div>

问题：

- 需要模拟复杂的对象图 (DatabaseManager → ConnectionPool → Connection)
- 类之间紧密耦合
- 难以隔离测试

</div>
</div>

---

# 缺陷 #2: 示例对比 (继续)

### 之后：可测试的设计

<div class="grid cols-2 gap-4">
<div>

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

</div>
<div>

优势：

- 清晰的单一依赖
- 易于模拟 UserRepository
- 遵循得墨忒耳定律

</div>
</div>

---

# 缺陷 #3: 脆弱的全局状态和单例

## 问题所在

::v-clicks

- 全局状态使代码不可预测
- 单例创建隐藏依赖
- 测试变得依赖顺序
- 难以并行运行测试
- 难以隔离被测系统

::

---

# 缺陷 #3: 脆弱的全局状态和单例 (继续)

## 警告信号及说明

- **添加或使用单例**

  ```cpp
  class UserService {
    public: void createUser(User user) {
      // 使用单例
      DatabaseConnection conn = DatabaseManager::getInstance().getConnection();
      conn.save(user);
    }
  };
  ```

  说明：单例模式隐藏了类的依赖关系，使得难以替换为测试替身，也使得测试之间相互影响。

---

- **添加或使用静态字段或静态方法**

  ```cpp
  class OrderService {
    private: static Cache cache;  // 静态字段

    public: Order getOrder(int id) {
      // 使用静态方法
      return CacheManager::getCachedOrder(id);
    }
  };
  ```

  说明：静态字段和方法创建了全局状态，使得测试之间相互影响，难以并行运行。

---

- **添加或使用静态初始化块**

  ```cpp
  class Logger {
    private: static Logger instance;

    // 静态初始化块
    static {
      instance = new Logger();
      instance.setLevel(LogLevel.INFO);
      instance.setFile("app.log");
    }
  };
  ```

  说明：静态初始化块使得类的行为在测试间难以控制和修改。

---

- **添加或使用注册表**

  ```cpp
  class PaymentService {
    public: void processPayment(Payment payment) {
      // 使用注册表
      PaymentProcessor processor = ServiceRegistry.get("paymentProcessor");
      processor.process(payment);
    }
  };
  ```

  说明：注册表和服务定位器隐藏了真实的依赖关系，使得难以理解类的实际需求。

---

- **添加或使用服务定位器**
  ```cpp
  class NotificationService {
    public: void sendNotification(Notification notification) {
      // 使用服务定位器
      EmailService emailService = ServiceLocator.getEmailService();
      emailService.send(notification);
    }
  };
  ```
  说明：服务定位器虽然比单例稍好，但仍然隐藏了依赖关系，不利于测试。

---

# 缺陷 #3: 示例

## 之前：难以测试

<div class="grid cols-2 gap-4">
<v-clicks>
<div>

```cpp
class OrderProcessor {
public:
  void processOrder(Order order) {
    // 使用全局状态和单例
    PaymentService
      .getInstance()
      .charge(order.getAmount());
    InventoryManager
      .getInstance()
      .updateStock(order.getItems());
    Logger
      .getLogger()
      .log("Order processed: " + order.getId());
  }
};
```

</div>
<div>

问题：

- 无法模拟单例实例
- 测试通过全局状态相互影响
- 难以轻松测试错误场景
- 难以并行运行测试

</div>
</v-clicks>
</div>

---

## 之后：可测试的设计

<div class="grid cols-2 gap-4">
<div>

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

</div>
<div>

优势：

- 依赖关系明确
- 易于注入模拟对象
- 无全局状态
- 测试可以独立运行

</div>
</div>

---

## 补充：替代单例模式的方法

### 1. 依赖注入 (Dependency Injection)

```cpp
class OrderProcessor {
private:
    PaymentService& paymentService;
    InventoryManager& inventoryManager;
    Logger& logger;

public:
    // 通过构造函数注入依赖
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

---

### 2. 工厂模式 (Factory Pattern)

```cpp
class ServiceFactory {
public:
    static PaymentService createPaymentService() {
        return PaymentService();
    }

    static InventoryManager createInventoryManager() {
        return InventoryManager();
    }
};
```

---

### 3. 控制反转 (Inversion of Control)

```cpp
// 定义接口
class IPaymentService {
public:
    virtual void charge(double amount) = 0;
};

// 具体实现
class StripePaymentService : public IPaymentService {
public:
    void charge(double amount) override {
        // Stripe 实现
    }
};

// 通过容器管理实例
class ServiceContainer {
private:
    std::unique_ptr<IPaymentService> paymentService;
public:
    ServiceContainer() {
        paymentService = std::make_unique<StripePaymentService>();
    }
    IPaymentService& getPaymentService() {
        return *paymentService;
    }
};
```

---

### 4. 参数传递

```cpp
class OrderProcessor {
public:
    // 将依赖作为参数传递
    void processOrder(Order order,
                     PaymentService& paymentService,
                     InventoryManager& inventoryManager) {
        paymentService.charge(order.getAmount());
        inventoryManager.updateStock(order.getItems());
    }
};
```

---

### 这些方法的优势：

- **可测试性**：可以轻松注入模拟对象进行测试
- **灵活性**：可以在运行时更改实现
- **松耦合**：类不依赖于具体实现
- **可维护性**：依赖关系明确，易于理解和修改

---

# 缺陷 #4: 类职责过多

## 问题所在

::v-clicks

- 具有多个职责的类难以理解
- 难以测试所有场景
- 一个职责的更改可能破坏其他职责
- 违反单一职责原则

::

---

# 缺陷 #4: 类职责过多 (继续)

## 警告信号及说明

- **总结类的作用时包含"和"字**

  ```cpp
  // 这个类方法的作用是验证用户、保存用户和发送欢迎邮件。
  class UserService {
    public:
      void registerUser(UserData data) {
        // 验证用户
        validateUserData(data);
        // 保存用户
        saveUser(data);
        // 发送邮件
        sendWelcomeEmail(data);
      }
  };
  ```

  说明：当需要用"和"来描述类的职责时，表明类承担了过多职责，应该拆分为多个专注的类。

---

- **新团队成员难以阅读并快速理解类的作用**

  这个类到底是做什么？

  ```cpp
  class OrderManager {
    // 包含太多方法，职责不清晰
    public:
      void createOrder() { /* ... */ } // 创建订单
      void calculateTax() { /* ... */ } // 计算税
      void generateInvoice() { /* ... */ } // 生成发票
      void sendConfirmation() { /* ... */ } // 发送确认邮件
      void updateInventory() { /* ... */ } // 更新库存
      void processPayment() { /* ... */ } // 处理支付
  };
  ```

  说明：类应该有清晰、专注的职责，使新成员能够快速理解其作用。

---

- **类中的字段只在某些方法中使用**

  是不是强行把多个字段放在同一个类中？

  ```cpp
  class ReportGenerator {
    private:
      EmailService emailService;     // 只在sendReport中使用
      FileService fileService;       // 只在saveReport中使用
      DatabaseService databaseService; // 只在loadData中使用

    public:
      void loadData() { /* 只使用databaseService */ }
      void saveReport() { /* 只使用fileService */ }
      void sendReport() { /* 只使用emailService */ }
  };
  ```

  说明：如果字段只在部分方法中使用，表明类可能承担了多个职责，应该拆分。

---

- **类中有只操作参数的静态方法**

  把工具类放进了一个有状态的类中。

  ```cpp
  class User {
    private:
      std::string firstName;
      std::string lastName;
      int age;

    // 静态方法只操作参数，与类的状态无关
    public:
      static bool isValidEmail(string email) { /* ... */ }
      static bool isAdult(int age) { /* ... */ }
      static string formatName(string firstName, string lastName) { /* ... */ }
  };
  ```

  说明：只操作参数的静态方法应该移到更合适的工具类中，或者成为相关类的实例方法。

---

# 缺陷 #4: 示例

<div grid="~ cols-2 gap-4">
<div>

### 之前：难以测试

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
};
```

</div>
<div>

问题：

- 多个职责：验证、持久化、通知
- 难以隔离测试
- 每个测试需要复杂的设置

</div>
</div>

---

### 之后：可测试的设计

<div grid="~ cols-2 gap-4">
<div>

```cpp
// 拆分为专注的类
class UserValidator {
public:
  bool isValidEmail(string email) { /* 只有验证逻辑 */ }
};

class UserRepository {
private:
  DatabaseConnection& conn;
public:
  UserRepository(DatabaseConnection& conn)
    : conn(conn) {}
  void save(User user) { conn.save(user); }
};

class WelcomeEmailSender {
private:
  EmailService& emailService;
public:
  WelcomeEmailSender(EmailService& emailService)
    : emailService(emailService) {}
  void sendWelcomeEmail(User user) {
    emailService.sendWelcomeEmail(user);
  }
};
```

</div>
<div>

优势：

- 每个类都有单一职责
- 易于单独测试每个类
- 依赖关系清晰

</div>
</div>

---

# 测试的好处

当我们消除这些缺陷后，测试变得容易得多：

::v-clicks

- **快速**：测试运行迅速，不依赖外部资源
- **隔离**：测试之间互不影响，可以独立运行
- **具体**：当测试失败时，我们知道确切的问题所在
- **清晰**：测试代码易于理解，表达明确的意图
- **可维护**：实现更改时测试不会中断，除非行为确实改变

::

---

# 总结

<div grid="~ cols-2 gap-4">
<div>

::v-clicks

1. **构造函数应该只将参数分配给字段**
   - 注入依赖而不是创建它们
   - 保持构造函数简洁

2. **直接请求依赖**
   - 不要深入了解协作者以获取其他对象
   - 遵循得墨忒耳定律

::

</div>
<div>

::v-clicks

3. **避免全局状态和单例**
   - 使依赖关系明确
   - 使用依赖注入

4. **遵循单一职责原则**
   - 一个类，一个职责
   - 保持类小而专注

::

</div>
</div>

---

> "防止 Bug 最有效的方法是编写可测试的代码。"
> -- Miško Hevery

::

---

# 参考资料

::v-clicks

- Miško Hevery的[编写可测试代码指南](https://github.com/mhevery/guide-to-testable-code)
- [Google测试博客](https://testing.googleblog.com/)
- Robert C. Martin的《代码整洁之道》
- Kent Beck的《测试驱动开发》

::

<br>
::v-click

<div class="text-center">

谢谢！

</div>
