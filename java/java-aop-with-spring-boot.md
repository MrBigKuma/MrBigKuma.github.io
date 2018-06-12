# Java AOP with Spring boot
_日本語版は下にある_
## Motivation
I need to implement Process tracking for a long-running process.
The problem is the implementation (algorithm) of the process is complicated enough to add any more tracking code inside.

It’s ok to add the tracking code into the implementation but when it comes to maintenance, new developers will face a hard time to understand the code. Leaving the algorithm alone is hard to understand itself.

Moreover, I see that there will be more “aspects” will be added to this algorithm. At least logging will be added.

This is s simplified version of the code:
```java
class Algorithm {
	private final Optimizer1 optimizer1;
	private final Optimizer2 optimizer2;
	
	void calculate(Source source) {
		// complicated logic
		optimizer1.execute(source);
		optimizer2.execute(source);
	}
}

class Optimizer1 {
	void execute(Source source) {
		// complicated logic
	}
}

class Optimizer2 {
	void execute(Source source) {
		// complicated logic
	}
}
```

## Aspect Oriented Programming (AOP)
AOP seems to be a good fit in my use case because it helps to separate my algorithm with progress tracking.
**Java**
In Java, the most popular ones that support AOP is AspectJ. Which uses a compiler to generate code that wraps the target methods.

I have to admit that it makes the code cleaner. The algorithm implementation code is just the same. I still can test them easily with the Unit test without mocking any “Progress Tracker”.

The thing I don’t like is the algorithm code I tested is not the **actual** code that is executed in production 😦 (I had a hard time for debugging when it comes to integration test). When compiling, the AspectJ compiler will inject its magic code to the algorithm.

The worst thing is that one developer may modify the input (e.g. `source`)  in AspectJ’s wrapper code. The poor algorithm developer will take a difficult time trying to figure out why his unit test pass but failing in production environment.

The good coding practice will prevent us adding source modification interception like that but when deadline come or a new developer just wants to make the work done. It is very tempting to hack the code to make the task done.

Therefore, It’s necessary to write the integration test to validate the code after integration.

## AspectJ with Spring Boot
To use AOP we need at least 2 things: a _Point cut_ - annotate code location where we will intercept. And an _Advice_, which define what should we do at the intercepted point.
To identify a point cut, AspectJ can use:
1. method name (e.g. `@Pointcut("execution(* com.xxxx.service.*Optimizer.*(..))")`)
2. or annotation (e.g.  `@Pointcut(“@annotation(com.xxxx.service.TrackOptimizerProgress)”)`)

① Good when there is a huge amount of code to track. It is also magic because other developers don’t know how can the method (e.g. `optimizer` ) be tracked.
② Good because it is explicit. But we have to annotate every time we use it.

Implementation with annotation
```java
class Algorithm {
	private final Optimizer1 optimizer1;
	private final Optimizer2 optimizer2;
	
	void calculate(Source source) {
		// complicated logic
		optimizer1.execute(source);
		optimizer2.execute(source);
	}
}

class Optimizer1 {
	@TrackOptimizerProgress
	public void execute(Source source) {
		// complicated logic
	}
}

class Optimizer2 {
	@TrackOptimizerProgress
	public void execute(Source source) {
		// complicated logic
	}
}

// ------ AspectJ code ------
// Define aspect annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TrackOptimizerProgress {}

// Define Advice
@Aspect
public class CalculationProgressAspect {
	@Pointcut(“@annotation(com.xxxx.service.TrackOptimizerProgress)”)
	public void optimizerExecuted() {}

	@Before("optimizerExecuted()")
	public void beforeOptimizer(JoinPoint joinPoint) throws Throwable {
		// Can do sth like getting Optimizer class name for tracking
		// calculationProgressTracker is another "aspect"
		calculationProgressTracker.startOptimize();
	}
}
```

The example code will execute `calculationProgressTracker.startOptimize()` before any Optimizer’s execution.

## Inject dependency to AspectJ with Spring Boot
I also define my different aspect (i.e. `CalculationProgressTracker`) as a bean. So I want to inject it to AspectJ’s advice. Here is the configuration:

```java
@Aspect
public class CalculationProgressAspect {
	@Autowired
	CalculationProgressTracker calculationProgressTracker;

	@Before("optimizerExecuted()")
	public void beforeOptimizer(JoinPoint joinPoint) throws Throwable {
		calculationProgressTracker.startOptimize();
	}
}
```

In Spring Boot `@Configuration` we define Aspect as a bean and adding `@EnableAspectJAutoProxy` for annotated optimizers:

```java
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy
public class ServiceProvider{
	@Bean
	public Optimizer1 optimizer1(){
		return new Optimizer1();
	}

	@Bean
	public CalculationProgressAspect calculationProgressAspect() {
 	   return new CalculationProgressAspect();
	}
}
```

Dependencies in grade:

```
dependencies {
...
	compile('org.springframework.boot:spring-boot-starter-aop')
}
```

## Reference:
1. [I want my AOP!, Part 1 | JavaWorld](https://www.javaworld.com/article/2073918/core-java/i-want-my-aop---part-1.html)
2. [Implementing AOP With Spring Boot and AspectJ - DZone Java](https://dzone.com/articles/implementing-aop-with-spring-boot-and-aspectj)
3. [Can DDD be Adequately Implemented Without DI and AOP?](https://www.infoq.com/news/2008/02/ddd-di-aop)
4. [oop - Aspect Oriented Programming vs. Object-Oriented Programming - Stack Overflow](https://stackoverflow.com/questions/232884/aspect-oriented-programming-vs-object-oriented-programming)
#blog

- - - -
**日本語版**
# Java AOPとSpring Boot
## モチベーション
私は時間がかかる実行プロセスの進捗トラッカーを実装させられました。

問題はプロセスの実装（アルゴリズム）が複雑ので進捗トラッカーのコードを入れるともっと複雑になります。

なお、進捗トラッカーだけではなくて間も無くロガーが追加の希望もある可能性が高いです。

これはアルゴリズムのコードの簡単版です。
```java
class Algorithm {
	private final Optimizer1 optimizer1;
	private final Optimizer2 optimizer2;
	
	void calculate(Source source) {
		// complicated logic
		optimizer1.execute(source);
		optimizer2.execute(source);
	}
}

class Optimizer1 {
	void execute(Source source) {
		// complicated logic
	}
}

class Optimizer2 {
	void execute(Source source) {
		// complicated logic
	}
}
```

## Aspect Oriented Programming (AOP)
この問題はAOPで解決のは一番いいだと思います。アルゴリズムと進捗トラッカーのコードがちゃんと分けられるですから。

**Java**
JavaでAOPが対応するFrameworkはAspectJと言うFrameworkです。AspectJはcompilerで狙いメソッドが包むコードを生成します。

AOPを使ったらアルゴリズムのコード何にも変わらないです。進捗トラッカーがモックせずにユニットテストが簡単で書けます。

気にすることはテストしたコードは実にproductionで実行するコードではないです（総合テストで時間かかったデバッグしたことがあります）。コンパイル時、AspectJのコンパイラがアルゴリズムのコードに魔法のコードを注入します。

「The worst thing is that one developer may modified the input (e.g. `source`)  in AspectJ’s wrapper code. The poor algorithm developer will take a difficult time try to figure out why his unit test pass but failing in production environment.

The good coding practice will prevent us adding source modification interception like that but when deadline come or a new developer just want to make the work done. It is every tempting to hack the code to make the task done.

Therefore, It’s necessary to write integration test to validate the code after integration.」

## AspectJとSpring boot
AOPを使うとせめて二つが必要です：
1. Point cut：AspectJのコードを追加するコードの所です。作成方法：
	* ① メソッドの名前。例： `@Pointcut("execution(* com.xxxx.service.*Optimizer.*(..))")`
	* ② アノテーション。例： `@Pointcut(“@annotation(com.xxxx.service.TrackOptimizerProgress)”)`
2. Advice：追加するコードです。

① 注入したい対象が多いと良いと思います。しかし、メンテナンスは大変だと思います。
② は明らかのに毎回使うとアノテーションを付けないといけません。

私は②で実装しました。
```java
class Algorithm {
	private final Optimizer1 optimizer1;
	private final Optimizer2 optimizer2;
	
	void calculate(Source source) {
		// 複雑ロジック
		optimizer1.execute(source);
		optimizer2.execute(source);
	}
}

class Optimizer1 {
	@TrackOptimizerProgress
	public void execute(Source source) {
		// 複雑ロジック
	}
}

class Optimizer2 {
	@TrackOptimizerProgress
	public void execute(Source source) {
		// 複雑ロジック
	}
}

// ------ AspectJ コード ------
// AspectJのためアノテーションを作成
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TrackOptimizerProgress {}

// AspectJのアドバイスを作成
@Aspect
public class CalculationProgressAspect {
	@Pointcut(“@annotation(com.xxxx.service.TrackOptimizerProgress)”)
	public void optimizerExecuted() {}

	@Before("optimizerExecuted()")
	public void beforeOptimizer(JoinPoint joinPoint) throws Throwable {
		// Can do sth like getting Optimizer class name for tracking
		// calculationProgressTracker is another "aspect"
		calculationProgressTracker.startOptimize();
	}
}
```

上のコードはオプティマイザの実行する前に`calculationProgressTracker.startOptimize()`を実行します。

## Spring BootでAspectJにDIする
`CalculationProgressTracker`がAspectJのアドバイスで注入するため、次の設定が必要です：

```java
@Aspect
public class CalculationProgressAspect {
	@Autowired
	CalculationProgressTracker calculationProgressTracker;

	@Before("optimizerExecuted()")
	public void beforeOptimizer(JoinPoint joinPoint) throws Throwable {
		calculationProgressTracker.startOptimize();
	}
}
```

Spring Bootの `@Configuration` で`@Bean`としてAspectを作成します。 アノテーションが付けたBeanも`@EnableAspectJAutoProxy` も必要です：

```java
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy
public class ServiceProvider{
	@Bean
	public Optimizer1 optimizer1(){
		return new Optimizer1();
	}

	@Bean
	public CalculationProgressAspect calculationProgressAspect() {
 	   return new CalculationProgressAspect();
	}
}
```

Gradeです:

```
dependencies {
...
	compile('org.springframework.boot:spring-boot-starter-aop')
}
```

## Reference:
1. [I want my AOP!, Part 1 | JavaWorld](https://www.javaworld.com/article/2073918/core-java/i-want-my-aop---part-1.html)
2. [Implementing AOP With Spring Boot and AspectJ - DZone Java](https://dzone.com/articles/implementing-aop-with-spring-boot-and-aspectj)
3. [Can DDD be Adequately Implemented Without DI and AOP?](https://www.infoq.com/news/2008/02/ddd-di-aop)
4. [oop - Aspect Oriented Programming vs. Object-Oriented Programming - Stack Overflow](https://stackoverflow.com/questions/232884/aspect-oriented-programming-vs-object-oriented-programming)