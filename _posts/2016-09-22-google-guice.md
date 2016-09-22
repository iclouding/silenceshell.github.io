---
layout: post
title: "Google Guice的动机"
date: 2016-09-22 00:11:12
author: 伊布
categories: tech
tags: Guice
cover:  "/assets/instacode.png"
---

原文地址: [Google Guice Motivation](https://github.com/google/guice/wiki/Motivation)。这篇文章捋了下Google Guice造轮子的思路，举了一个很具体的例子，~~九浅一深~~深入浅出，对理解为什么要有依赖注入很有帮助。
Google Guice是一个类似Spring的DI框架，优点是简单，轻量级，快。Google Guice和Spring的对比可以参考[SpringComparison](https://github.com/google/guice/wiki/SpringComparison)和[这篇文章](http://blog.csdn.net/derekjiang/article/details/7213829)。


---

### 动机

应用开发时，所有代码都堆在一起是很乏味的。数据、服务、展示类之间的互联，有多种方式。为了对比，我们写一个披萨订购网站的计费代码。

```java
public interface BillingService {

  /**
   * Attempts to charge the order to the credit card. Both successful and
   * failed transactions will be recorded.
   *
   * @return a receipt of the transaction. If the charge was successful, the
   *      receipt will be successful. Otherwise, the receipt will contain a
   *      decline note describing why the charge failed.
   */
  Receipt chargeOrder(PizzaOrder order, CreditCard creditCard);
}
```

接口实现完了以后，我们还需要对其做单元测试；在单元测试中，我们需要一个FakeCreditCardProcessor来避免发生真实的信用卡消费。


### 直接构造函数调用(Direct constructor calls)

直接new信用卡处理。

```java
public class RealBillingService implements BillingService {
  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    CreditCardProcessor processor = new PaypalCreditCardProcessor();
    TransactionLog transactionLog = new DatabaseTransactionLog();

    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```

这块代码的模块化和可测试性比较差。而且做单元测试的时候，会调用真实的Processor，发生信用卡消费。另外也不方便测试信用卡被拒或者服务不可用的情况（无法改变Process的行为）。

### Factories

工厂类可以把客户端跟实现类解耦掉。简单的工厂类使用静态方法来获取和设置接口的mock实现。样板代码：

```java
public class CreditCardProcessorFactory {

  private static CreditCardProcessor instance;

  public static void setInstance(CreditCardProcessor processor) {
    instance = processor;
  }

  public static CreditCardProcessor getInstance() {
    if (instance == null) {
      return new SquareCreditCardProcessor();
    }

    return instance;
  }
}
```

客户端这边用 factory lookups 代替了new方法：

```java
public class RealBillingService implements BillingService {
  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    CreditCardProcessor processor = CreditCardProcessorFactory.getInstance();
    TransactionLog transactionLog = TransactionLogFactory.getInstance();

    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```

fatory让单元测试成为可能(提前用桩去setInstance)：

```java
public class RealBillingServiceTest extends TestCase {

  private final PizzaOrder order = new PizzaOrder(100);
  private final CreditCard creditCard = new CreditCard("1234", 11, 2010);

  private final InMemoryTransactionLog transactionLog = new InMemoryTransactionLog();
  private final FakeCreditCardProcessor processor = new FakeCreditCardProcessor();

  @Override public void setUp() {
    TransactionLogFactory.setInstance(transactionLog);
    CreditCardProcessorFactory.setInstance(processor);
  }

  @Override public void tearDown() {
    TransactionLogFactory.setInstance(null);
    CreditCardProcessorFactory.setInstance(null);
  }

  public void testSuccessfulCharge() {
    RealBillingService billingService = new RealBillingService();
    Receipt receipt = billingService.chargeOrder(order, creditCard);

    assertTrue(receipt.hasSuccessfulCharge());
    assertEquals(100, receipt.getAmountOfCharge());
    assertEquals(creditCard, processor.getCardOfOnlyCharge());
    assertEquals(100, processor.getAmountOfOnlyCharge());
    assertTrue(transactionLog.wasSuccessLogged());
  }
}
```

虽然可以做单元测试了，但是上面这块代码还是比较笨拙。由于是用全局变量（指工厂类的instance）来保存模拟实现，所以需要小心setup()和tearDown()。万一tearDown()的时候失败了，全局变量仍然指向测试实例，会影响后面的单元测试。另外，全局变量也会导致单元测试无法并行执行。

但最大的问题是，依赖是隐藏在代码里的。如果我们在RealBillingService里加了新依赖CreditCardFraudTracker，就需要把单元测试重新跑一遍(因为从单元测试这里看不到需要哪些Factory)。另，万一我们忘了在生产环境上初始化实例，那么只有到要信用卡收费了才会发现，编译器不会帮我们发现。保姆式的工厂类会变得越来越头疼。

虽然可以通过充分测试来避免，我们有更好、更有效的办法。

### 依赖注入

跟Factory一样，依赖注入也是一种设计模式，其核心原则是*将行为从依赖解析中分离出来*。在我们的例子里，RealBillingService并不需要去找TransactionLog 和 CreditCardProcessor这两个类，因为这两个类会作为构造参数传进来。换句话说，就是把锅甩给了使用RealBillingService的client。

```java
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processor;
  private final TransactionLog transactionLog;

  public RealBillingService(CreditCardProcessor processor,
      TransactionLog transactionLog) {
    this.processor = processor;
    this.transactionLog = transactionLog;
  }

  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```

这样就不需要factory，单元测试也不需要setUp和tearDown。

```java
public class RealBillingServiceTest extends TestCase {

  private final PizzaOrder order = new PizzaOrder(100);
  private final CreditCard creditCard = new CreditCard("1234", 11, 2010);

  private final InMemoryTransactionLog transactionLog = new InMemoryTransactionLog();
  private final FakeCreditCardProcessor processor = new FakeCreditCardProcessor();

  public void testSuccessfulCharge() {
    RealBillingService billingService
        = new RealBillingService(processor, transactionLog);
    Receipt receipt = billingService.chargeOrder(order, creditCard);

    assertTrue(receipt.hasSuccessfulCharge());
    assertEquals(100, receipt.getAmountOfCharge());
    assertEquals(creditCard, processor.getCardOfOnlyCharge());
    assertEquals(100, processor.getAmountOfOnlyCharge());
    assertTrue(transactionLog.wasSuccessLogged());
  }
}
```

现在，无论是添加还是删除依赖关系，编译器都会提醒我们哪些测试例需要更新。API签名向外暴漏了依赖关系。

不过，如前面所说，锅现在给BillingService的client来背了，它需要去查依赖关系。我们可以用依赖注入模式来解决：依赖client的类，可以把BillingService也作为构造参数（这块没有看的很明白）。顶级类需要个框架支持，否则就需要像下面这样去递归构造依赖了(RealBillingService里已经构造过一遍了)：

```java
public static void main(String[] args) {
  CreditCardProcessor processor = new PaypalCreditCardProcessor();
  TransactionLog transactionLog = new DatabaseTransactionLog();
  BillingService billingService
      = new RealBillingService(processor, transactionLog);
  ...
}
```

### Dependency Injection with Guice

依赖注入让代码模块化、可测试，而在Guice的帮助下，代码会更容易编写。我们用Guice改写下这个订单系统。

首先需要告诉Guice是接口和实现的映射关系。在Guice module里实现这个配置（Guice module是一个实现Module接口的java类）。相对Spring的xml定义方式来说，这种module方式更为直接，也更好理解(我实在是很烦xml)。

```java
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(TransactionLog.class).to(DatabaseTransactionLog.class);
    bind(CreditCardProcessor.class).to(PaypalCreditCardProcessor.class);
    bind(BillingService.class).to(RealBillingService.class);
  }
}
```

然后给RealBillingService的构造器加一个@Inject注解，从而让Guice去查找每个参数的依赖关系(上面已经绑定了接口和类的映射关系，Guice据此可以找到实现类)。

```java
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processor;
  private final TransactionLog transactionLog;

  @Inject
  public RealBillingService(CreditCardProcessor processor,
      TransactionLog transactionLog) {
    this.processor = processor;
    this.transactionLog = transactionLog;
  }

  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```

OK，收工。Injector可以用来获取任何前面map过的类的实例。

```java
public static void main(String[] args) {
  Injector injector = Guice.createInjector(new BillingModule());
  BillingService billingService = injector.getInstance(BillingService.class);
  ...
}
```
