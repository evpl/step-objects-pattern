# _Step objects_ and _Step objects chain_ patterns

## Table of Contents

* [1. _Step objects_ pattern](#1.-_Step-objects_-pattern)
    * [1.1. Using](#1.1.-Using)
    * [1.2. Definition](#1.2.-Definition)
    * [1.3. Code example](#1.3.-Code-example)
* [2. _Step objects chain_ pattern](#2.-_Step-objects-chain_-pattern)
    * [2.1. Using](#2.1.-Using)
    * [2.2. Definition](#2.2.-Definition)
    * [2.3.Code example](#2.3.-Code-example)
* [3. Using for automated testing](#3.-Using-for-automated-testing)
    * [3.1. Logging](#3.1.-Logging)
    * [3.2. Page object replacement](#3.2.-Page-object-replacement)
* [4. Implementations](#4.-Implementations)

## 1. _Step objects_ pattern

### 1.1. Using

This pattern can be used both for software development and for automated testing development. The examples are given
in Java, but the pattern is also suitable for other languages such as Kotlin, C#, etc.

### 1.2. Definition

This pattern allows you to reuse code by splitting it into different classes.

1. The class must contains constructors for setting parameters and a single method to perform an action. For Java,
   you can take Java Functional interfaces like the `Consumer`, `Function`, `Supplier` and `Runnable` as a basis
   (and also `BiConsumer`, `BiFunction`, etc.).
2. You can compose steps of any level of complexity.
3. You can combine steps to compose complex steps.
4. You can also set additional behavior using decorators and annotations.

### 1.3. Code example

We have this code.

```java
int expectedStringLength = 10;
byte[] array = new byte[expectedStringLength];
new Random().nextBytes(array);
String randomString = new String(array, StandardCharsets.UTF_8);
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    throw new RuntimeException(e);
}
int actualStringLength = randomString.length();
if (actualStringLength != expectedStringLength) {
    throw new AssertionError();
}
```

Let's try to logically split it into separate steps.

```java
/* Step 1 - random string generation */
int expectedStringLength = 10;
byte[] array = new byte[expectedStringLength];
new Random().nextBytes(array);
String randomString = new String(array, StandardCharsets.UTF_8);
/* Step 2 - waiting */
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    throw new RuntimeException(e);
}
/* Step 3 - getting the length of a string */
int actualStringLength = randomString.length();
/* Step 4 - assertion */
if (actualStringLength != expectedStringLength) {
    throw new AssertionError();
}
```

And then we put these steps into separate classes using standard Java functional interfaces.

```java
public class RandomString implements Supplier<String> {
    private final int length;

    public RandomString(int length) {
        this.length = length;
    }

    @Override
    public String get() {
        byte[] array = new byte[this.length];
        new Random().nextBytes(array);
        return new String(array, StandardCharsets.UTF_8);
    }
}

public class Wait implements Runnable {
    private final int millis;

    public Wait(int millis) {
        this.millis = millis;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(this.millis);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}

public class GetStringLength implements Function<String, Integer> {

    public GetStringLength() {
    }

    @Override
    public Integer apply(String string) {
        return string.length();
    }
}

public class AssertEquals<T> implements Consumer<T> {
    private final T expected;

    public AssertEquals(T expected) {
        this.expected = expected;
    }

    @Override
    public void accept(T actual) {
        if (Objects.equals(actual, this.expected)) {
            throw new AssertionError();
        }
    }
}
```

Let's rewrite the original code. As you can see now the code is more readable even without comments because we can
see the actions by the names of the steps.

```java
int expectedStringLength = 10;
String randomString = new RandomString(expectedStringLength).get();
new Wait(1000).run();
int actualStringLength = new GetStringLength().apply(randomString);
new AssertEquals<>(expectedStringLength).accept(actualStringLength);
```

## 2. _Step objects chain_ pattern

### 2.1. Using

This pattern can also be used both for software development and for automated testing development. The examples are
given in Java, but the pattern is also suitable for other languages such as Kotlin, C#, etc.

### 2.2. Definition

Step objects pattern is extended by the step executor class. This pattern allows you to execute step objects in a more
readable and logical way. The executor should allow you to perform step objects actions in chain. Additional behavior
can be added to the executor such as error handling.

### 2.3. Code example

Let's take the origin code and step objects from the code example of
[_Step objects_ pattern](#1.-_Step-objects_-pattern).

Let's write an executor that saves some context.

```java
public class StepExecutor<T> {
    private final T context;

    public StepExecutor() {
        this(null);
    }

    public StepExecutor(T context) {
        this.context = context;
    }

    public StepExecutor<T> exec(Runnable runnable) {
        runnable.run();
        return this;
    }

    public StepExecutor<T> exec(Consumer<? super T> consumer) {
        consumer.accept(this.context);
        return this;
    }

    public <R> StepExecutor<R> exec(Supplier<? extends R> supplier) {
        return new StepExecutor<>(supplier.get());
    }

    public <R> StepExecutor<R> exec(Function<? super T, ? extends R> function) {
        return new StepExecutor<>(function.apply(this.context));
    }
}
```

Now step objects can be used in a chain.

```java
int expectedStringLength = 10;
new StepExecutor<>()
    .exec(new RandomString(expectedStringLength))
    .exec(new Wait(1000))
    .exec(new GetStringLength())
    .exec(new AssertEquals<>(expectedStringLength));
```

## 3. Using for automated testing

### 3.1. Logging

You can use step objects to log steps in a reporting system.

Using static methods (in style of [Allure](https://github.com/allure-framework/allure-java) or
[Xteps](https://github.com/evpl/xteps)).

```java
public class StepObject implements Runnable {
    
    public StepObject() {
    }
    
    @Override
    public void run() {
        step("Step name", () -> {
            //...
        });
    }
}
```

Or with annotations (in style of [Allure](https://github.com/allure-framework/allure-java) or
[ReportPortal](https://github.com/reportportal/client-java)).

```java
public class StepObject implements Runnable {

    public StepObject() {
    }

    @Override
    @Step("Step name")
    public void run() {
        //...
    }
}
```

### 3.2. Page object replacement

Both patterns can be used not only as a way to organize code but also as a replacement for the bulky
Page Object pattern.

Let's write step objects using the [Selenium WebDriver](https://github.com/SeleniumHQ/selenium).

```java
public class CreateWebDriver implements Supplier<WebDriver> {

    public CreateWebDriver() {
    }

    @Override
    public WebDriver get() {
        return new ChromeDriver();
    }
}

public class Open implements Consumer<WebDriver> {
    private final String url;

    public Open(String url) {
        this.url = url;
    }

    @Override
    public void accept(WebDriver webDriver) {
        webDriver.navigate().to(this.url);
    }
}

public class TypeLogin implements Consumer<WebDriver> {
    private final String login;

    public TypeLogin(String login) {
        this.login = login;
    }

    @Override
    public void accept(WebDriver webDriver) {
        webDriver.findElement(By.id("username")).sendKeys(this.login);
    }
}

public class TypePassword implements Consumer<WebDriver> {
    private final String password;

    public TypePassword(String password) {
        this.password = password;
    }

    @Override
    public void accept(WebDriver webDriver) {
        webDriver.findElement(By.id("username")).sendKeys(this.password);
    }
}

public class ClickOnLoginButton implements Consumer<WebDriver> {

    public ClickOnLoginButton() {
    }

    @Override
    public void accept(WebDriver webDriver) {
        webDriver.findElement(By.id("sign_in")).click();
    }
}

public class LoginAs implements Consumer<WebDriver> {
    private final String login;
    private final String password;

    public LoginAs(String login, String password) {
        this.login = login;
        this.password = password;
    }

    @Override
    public void accept(WebDriver webDriver) {
        new TypeLogin(this.login).accept(webDriver);
        new TypePassword(this.password).accept(webDriver);
        new ClickOnLoginButton().accept(webDriver);
    }
}

public class AssertEquals<T> implements Consumer<T> {
    private final T expected;

    public AssertEquals(T expected) {
        this.expected = expected;
    }

    @Override
    public void accept(T actual) {
        if (Objects.equals(actual, this.expected)) {
            throw new AssertionError();
        }
    }
}
```

And write a chain of actions.

```java
new StepExecutor<>()
    .exec(new CreateWebDriver())
    .exec(new Open("https://...com"))
    .exec(new LoginAs("user1", "123123"))
    .exec(new GetTitle())
    .exec(new AssertEquals<>("Expected title value"));
```

## 4. Implementations

At the moment the only library that allows you to use the _Step objects chain_ pattern is
[Xteps](https://github.com/evpl/xteps).

The above code with Xteps would look like this.

```java
stepsChain()
    .stepToContext(new CreateWebDriver())
    .step(new Open("https://...com"))
    .step(new LoginAs("user1", "123123"))
    .stepToContext(new GetTitle())
    .step(new AssertEquals<>("Expected title value"));
```
