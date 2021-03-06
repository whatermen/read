# Transaction Management

- 01 [introduction-to-spring-framework-transaction-management](#introduction-to-spring-framework-transaction-management)
- 02 [global-transactions](#global-transactions)
- 03 [local-transactions](#local-transactions)
- 04 [understanding-the-spring-framework-transaction-abstraction](#understanding-the-spring-framework-transaction-abstraction)
- 05 [transactiondefinition](#transactiondefinition)
- 06 [transactionstatus](#transactionstatus)
- 07 [datasourcetransactionmanager](#datasourcetransactionmanager)
- 08 [declarative-transaction-management](#declarative-transaction-management)
- 09 [understanding-the-spring-frameworks-declarative-transaction-implementation](#understanding-the-spring-frameworks-declarative-transaction-implementation)
- 10 [example-of-declarative-transaction-implementation](#example-of-declarative-transaction-implementation)
- 11 [rolling-back-a-declarative-transaction](#rolling-back-a-declarative-transaction)
- 12 [configuring-different-transactional-semantics-for-different-beans](#configuring-different-transactional-semantics-for-different-beans)
- 13 [tx-advice-settings](#tx-advice-settings)
- 14 [transactional](#transactional)
- 15 [transactional-settings](#transactional-settings)
- 16 [transaction-propagation](#transaction-propagation)
- 17 [required](#required)
- 18 [requiresnew](#requiresnew)
- 19 [advising-transactional-operations](#advising-transactional-operations)
- 20 [programmatic-transaction-management](#programmatic-transaction-management)

## Introduction to Spring Framework transaction management

Consistent programming model across different transaction APIs such as Java Transaction API (JTA), JDBC, Hibernate, Java Persistence API (JPA), and Java Data Objects (JDO).
Support for declarative transaction management.
Simpler API for programmatic transaction management than complex transaction APIs such as JTA.
Excellent integration with Spring’s data access abstractions.

- Consistent programming model across different transaction APIs such as Java Transaction API (JTA), JDBC, Hibernate, Java Persistence API (JPA), and Java Data Objects (JDO).
- Support for declarative transaction management.
- Simpler API for programmatic transaction management than complex transaction APIs such as JTA.
- Excellent integration with Spring’s data access abstractions.

## Global transactions

Global transactions enable you to work with multiple transactional resources, typically relational databases and message queues

## Local transactions

Local transactions are resource-specific, such as a transaction associated with a JDBC connection. Local transactions may be easier to use, but have significant disadvantages: they cannot work across multiple transactional resources. For example, code that manages transactions using a JDBC connection cannot run within a global JTA transaction. Because the application server is not involved in transaction management, it cannot help ensure correctness across multiple resources. (It is worth noting that most applications use a single transaction resource.) Another downside is that local transactions are invasive to the programming model.

## Understanding the Spring Framework transaction abstraction

The key to the Spring transaction abstraction is the notion of a transaction strategy. A transaction strategy is defined by the `org.springframework.transaction.PlatformTransactionManager` interface:

[Link](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/htmlsingle/#transaction-strategies)

```java
public interface PlatformTransactionManager {

    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;

    void commit(TransactionStatus status) throws TransactionException;

    void rollback(TransactionStatus status) throws TransactionException;
}
```

## TransactionDefinition

[Link](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/htmlsingle/#transaction-strategies)

The TransactionDefinition interface specifies:

- Isolation: The degree to which this transaction is isolated from the work of other transactions. For example, can this transaction see uncommitted writes from other transactions?
- Propagation: Typically, all code executed within a transaction scope will run in that transaction. However, you have the option of specifying the behavior in the event that a transactional method is executed when a transaction context already exists. For example, code can continue running in the existing transaction (the common case); or the existing transaction can be suspended and a new transaction created. Spring offers all of the transaction propagation options familiar from EJB CMT. To read about the semantics of transaction propagation in Spring, see Section 17.5.7, “Transaction propagation”.
- Timeout: How long this transaction runs before timing out and being rolled back automatically by the underlying transaction infrastructure.
- Read-only status: A read-only transaction can be used when your code reads but does not modify data. Read-only transactions can be a useful optimization in some cases, such as when you are using Hibernate.

## TransactionStatus

[Link](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/htmlsingle/#transaction-strategies)

```java
public interface TransactionStatus extends SavepointManager {

    boolean isNewTransaction();

    boolean hasSavepoint();

    void setRollbackOnly();

    boolean isRollbackOnly();

    void flush();

    boolean isCompleted();

}
```

## DataSourceTransactionManager

PlatformTransactionManager implementations normally require knowledge of the environment in which they work: JDBC, JTA, Hibernate, and so on. The following examples show how you can define a local PlatformTransactionManager implementation. (This example works with plain JDBC.)

You define a JDBC DataSource

```xml
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="${jdbc.driverClassName}" />
    <property name="url" value="${jdbc.url}" />
    <property name="username" value="${jdbc.username}" />
    <property name="password" value="${jdbc.password}" />
</bean>
```

The related PlatformTransactionManager bean definition will then have a reference to the DataSource definition. It will look like this:

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

## Declarative transaction management

The concept of rollback rules is important: they enable you to specify which exceptions (and throwables) should cause automatic rollback. You specify this declaratively, in configuration, not in Java code. So, although you can still call setRollbackOnly() on the TransactionStatus object to roll back the current transaction back, most often you can specify a rule that MyApplicationException must always result in rollback. The significant advantage to this option is that business objects do not depend on the transaction infrastructure. For example, they typically do not need to import Spring transaction APIs or other Spring APIs.

Although EJB container default behavior automatically rolls back the transaction on a system exception (usually a runtime exception), EJB CMT does not roll back the transaction automatically on anapplication exception (that is, a checked exception other than java.rmi.RemoteException). While the Spring default behavior for declarative transaction management follows EJB convention (roll back is automatic only on unchecked exceptions), it is often useful to customize this behavior.

## Understanding the Spring Frameworks declarative transaction implementation

The most important concepts to grasp with regard to the Spring Framework’s declarative transaction support are that this support is enabled via AOP proxies, and that the transactional advice is driven by metadata (currently XML- or annotation-based). The combination of AOP with transactional metadata yields an AOP proxy that uses a `TransactionInterceptor` in conjunction with an appropriate `PlatformTransactionManager` implementation to drive transactions around method invocations.

![tx](./images/tx.png)

## Example of declarative transaction implementation

```xml
<!-- from the file 'context.xml' -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- this is the service object that we want to make transactional -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- the transactional advice (what 'happens'; see the <aop:advisor/> bean below) -->
    <tx:advice id="txAdvice" transaction-manager="txManager">
        <!-- the transactional semantics... -->
        <tx:attributes>
            <!-- all methods starting with 'get' are read-only -->
            <tx:method name="get*" read-only="true"/>
            <!-- other methods use the default transaction settings (see below) -->
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <!-- ensure that the above transactional advice runs for any execution
        of an operation defined by the FooService interface -->
    <aop:config>
        <aop:pointcut id="fooServiceOperation" expression="execution(* x.y.service.FooService.*(..))"/>
        <aop:advisor advice-ref="txAdvice" pointcut-ref="fooServiceOperation"/>
    </aop:config>

    <!-- don't forget the DataSource -->
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="oracle.jdbc.driver.OracleDriver"/>
        <property name="url" value="jdbc:oracle:thin:@rj-t42:1521:elvis"/>
        <property name="username" value="scott"/>
        <property name="password" value="tiger"/>
    </bean>

    <!-- similarly, don't forget the PlatformTransactionManager -->
    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!-- other <bean/> definitions here -->

</beans>
```

## Rolling back a declarative transaction

In its default configuration, the Spring Framework’s transaction infrastructure code only marks a transaction for rollback in the case of runtime, unchecked exceptions; that is, when the thrown exception is an instance or subclass of `RuntimeException`. ( `Errors` will also - by default - result in a rollback). Checked exceptions that are thrown from a transactional method do not result in rollback in the default configuration.

You can configure exactly which Exception types mark a transaction for rollback, including checked exceptions. The following XML snippet demonstrates how you configure rollback for a checked, application-specific Exception type.

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
    <tx:method name="get*" read-only="true" rollback-for="NoProductInStockException"/>
    <tx:method name="*"/>
    </tx:attributes>
</tx:advice>

<tx:advice id="txAdvice">
    <tx:attributes>
    <tx:method name="updateStock" no-rollback-for="InstrumentNotFoundException"/>
    <tx:method name="*"/>
    </tx:attributes>
</tx:advice>
```

## Configuring different transactional semantics for different beans

## tx-advice-settings

[Link](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/htmlsingle/#transaction-declarative-txadvice-settings)

- Propagation setting is REQUIRED.
- Isolation level is DEFAULT.
- Transaction is read/write.
- Transaction timeout defaults to the default timeout of the underlying transaction system, or none if timeouts are not supported.
- Any RuntimeException triggers rollback, and any checked Exception does not.

![](images/tx-methond.png)

## @Transactional

[Link](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/htmlsingle/#transaction-declarative-annotations)

配置：

```xml
 <!-- this is the service object that we want to make transactional -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- enable the configuration of transactional behavior based on annotations -->
    <tx:annotation-driven transaction-manager="txManager"/><!-- a PlatformTransactionManager is still required -->
    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!-- (this dependency is defined somewhere else) -->
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!-- other <bean/> definitions here -->
```

注意点：

- @Transactional 注解修饰的方法必须是`public`的
- In proxy mode (which is the default), only external method calls coming in through the proxy are intercepted (like `@PostConstruct`)
- Spring recommends that you only annotate concrete classes (and methods of concrete classes) with the @Transactional annotation, as opposed to annotating interfaces. You certainly can place the @Transactional annotation on an interface (or an interface method), but this works only as you would expect it to if you are using interface-based proxies. The fact that Java annotations are not inherited from interfaces means that if you are using class-based proxies ( proxy-target-class="true") or the weaving-based aspect ( mode="aspectj"), then the transaction settings are not recognized by the proxying and weaving infrastructure, and the object will not be wrapped in a transactional proxy, which would be decidedly bad.

## @Transactional settings

[Link](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/htmlsingle/#transaction-declarative-attransactional-settings)

The @Transactional annotation is metadata that specifies that an interface, class, or method must have transactional semantics; for example, "start a brand new read-only transaction when this method is invoked, suspending any existing transaction". The default @Transactional settings are as follows:

- Propagation setting is PROPAGATION_REQUIRED.
- Isolation level is ISOLATION_DEFAULT.
- Transaction is read/write.
- Transaction timeout defaults to the default timeout of the underlying transaction system, or to none if timeouts are not supported.
- Any RuntimeException triggers rollback, and any checked Exception does not.

![transaactional-settings](images/transaactional-settings.png)

## Transaction propagation

事物的传播属性

### Required

PROPAGATION_REQUIRED

![tx_prop_required](images/tx_prop_required.png)

### RequiresNew

PROPAGATION_REQUIRES_NEW

![tx_prop_requires_new](images/tx_prop_requires_new.png)

> PROPAGATION_REQUIRES_NEW, in contrast to PROPAGATION_REQUIRED, uses a completely independent transaction for each affected transaction scope. In that case, the underlying physical transactions are different and hence can commit or roll back independently, with an outer transaction not affected by an inner transaction’s rollback status.

## Advising transactional operations

通知 + 事务（顺序可控） 实现`Ordered`接口

[Link](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/htmlsingle/#transaction-declarative-applying-more-than-just-tx-advice)

## Programmatic transaction management

- Using the TransactionTemplate. [Link](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/htmlsingle/#tx-prog-template)
- Using a PlatformTransactionManager implementation directly. [Link](https://docs.spring.io/spring/docs/4.3.x/spring-framework-reference/htmlsingle/#transaction-programmatic-ptm)