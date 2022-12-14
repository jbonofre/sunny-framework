# Apache Sunny Framework

Apache Sunny Framework aims at providing a very light IoC container.

## Core Values

For cloud applications, being the most reactive possible is a key criteria. So Sunny Framework chose to:

* **Be build time oriented**: only delegate to runtime the bean resolution (to enable dynamic module aggregation) and not the bean and model discovery nor proxy generation.
* **Stay flexible**: even if the model is generated at build time, you can generally still customize it by removing a bean and adding your own one.
* **Be native friendly**: some applications need to be native to start very fast and bypass the classloading and a bunch of JVM init, for that purpose we ensure the IoC is GraalVM friendly.

## Features

* Field/constructor injections
* Contexts/scopes support
  * Default scope (`@DefaultScoped`) is to create an instance per lookup/injection
  * Application scope (`@ApplicationScoped`) creates an instance per container and instantiates it lazily (at first call)
* Event bus
  * `Start`/`Stop` events are fired with container related lifecycle hooks
* Lifecycle: `@Init`/`@Destroy` to react to the bean lifecycle
* `Optional<MyClass>` injections
* Basic cloud friendly configuration

### No interceptor support

Since few years, we got used to see declarative interceptors (annotations) like this:

```java
public class MyBean {
    @Traced
    public void doSomething() {
      // ...
    }
}
```

This is great and the container is actually linking the annotation to an implementation (a bean in general) which intercepts the call. This is not bad but has some design pitfalls:

* Most interceptors will use parameters and for such a generic approach to work, it needs an `Object[]` (or `List`) of parameters. This is really not fast (it requires to allocate an array for that purpose).
* It requires to know and understand the rules between class interceptors, method interceptors, appending/overriding when relevant plus the same with parent classes. All that can quickly become complex.
* It is often static: once put on a method, disabling an interceptor requires the underlying library to be able to do that or to use some advanced customization at startup to do it.

For those reasons, we think that we don't need an interceptor solution in Sunny Framework but we don't say the underlying feature is pointless. However, thanks to a more modern programming style, we can use functional approach to solve the same problem.
Therefore, previous example would rather become:

```java
public class MyBean {
    public void doSomething() {
        tracing(() -> {
            // ...
        });
    }
}
```

The big advantage is you can static utility if you want, but also rely on beans and even combine more efficiently interceptions in a custom and configurable fashion:

```java
public class MyBean {
    public void doSomething() {
        tracing(() -> timed(() -> logged(() -> {
            // ...
        })));
    }
}
```

can become:

```java
public class MyBean {
    @Injection
    MyObjeervabilityService obs;
    
    public void doSomething() {
        obs.instrumented(() -> {
            // ...
        });
    }
}
```

If you compare the case with parameters it is way more efficient in general since you just do a standard parameter passing call:

```java
public class MyBean {
    @Injection
    EntityManager em; // assume the application uses JPA - not required
  
    @Injection
    JpaService jpa; // custom bean to handle transaction for example
  
    public void store(final Transaction tx) {
        tracing(
                // no Object[] created for an interceptor
                // and no reflection to extract the id
                tx.id(),
                () -> jpa.tx(() -> em.persist(tx)));
    }
}
```

However, going with this approach can get the __chaining lambda__ pitfall. To solve this, we encourage you to ensure your "interceptor" can be chained properly using the same kind of callback.

Here is an example (the important part is more the signature than the fact it is a `static` method or bean method):

```java
public static <T> Supplier<T> interceptor1(String marker, Map<String, String> data, Supplier<T> nested) {
   return() -> {
      logger.info(message(marker,data)); // interceptor role
      return task.get(); // intercepted business, "ic.proceed()" in Jakarta interceptor API  
   };
}

public static <T> Supplier<T> interceptor12(Params params, Supplier<T> nested) {
    // same kind of logic of the implementation
}
```

Thanks to this definition which commonly agreed to use `Supplier<T>` as the intercepted call and the fact interceptor methods return a call and not execute it directly, you can chain them more easily:

```java
public void storeCustomer(final Customer customer) {
    interceptor2(
       Params.of(customer),
       interceptor1(
           "incoming-customer", Map.of("id", customer.id()),
           () -> {
                // business code
        })).get(); // trigger the actual execution, it is the terminal operation of the chain
}
```

If you want to go further you can use `Stream` to represent that. Now an interceptor is a `Function<Supplier<T>, Supplier<T>>` so if you define the list of interceptors in a `Stream`, you can just reduce them using the business logic as identity to have the actual invocation and execute it.
Only detail to take care: ensure to reverse the stream to call the interceptor in order:

```java
public void storeCustomer(final Customer customer) {
    Stream.<Function<Supplier<Void>, Supplier<Void>>>of(
                // reversed chain of interceptor (i1 will be executed before i2)
                delegate -> interceptor2(Params.of(customer), delegate),
                delegate -> interceptor1("incoming-customer", Map.of("id", customer.id()), delegate)
        )
        // merge the stream of interceptors as one execution wrapper
        .reduce(identity(), Function::andThen)
        .apply(() -> { // apply to the actual business logic
            System.out.println(">Business");
            return null;
        })
        .get(); // execute it
}
```

Indeed, in practice, you can extract that kind of code in an utility and use something like:

```java
// utility
public static <T> T intercepted(final Supplier<T> execution, final Function<Supplier<T>, Supplier<T>>... interceptors) {
    return Stream.of(interceptors)
            .reduce(identity(), Function::andThen)
            .apply(execution)
            .get();
}

// usage
intercepted(
    () -> { // business logic
        System.out.println(">Business");
        return null;
    },
    // interceptors
    delegate -> interceptor2(Params.of(customer), delegate),
    delegate -> interceptor1("incoming-customer", Map.of("id", customer.id()), delegate)
);
```

This is what the class `org.apache.sunny.ioc.api.composable.Wraps` does.

It's also possible to use `CompletionStage` with your interceptor, to add some behavior before/after the call even if the result is not computed synchronously.

## Limitations

---
**NOTE**

These limitations are the ones from today, none are technically strong limitations and Sunny Framework will improve.
---

* A no-arg constructor must be available for any class bean
* If a method producer bean is `Autocloseable` then it will be automatically closed
* Even methods can not be packaged scope if the enclosing bean uses a subclass proxy (like `@ApplicationScoped` context)
* Constructor injections are supported but for proxied scope (`@ApplicationScoped` for example), it requires a default no-arg constructor (in scope `protected` or `public`) in the class
* Event bus listeners can only have the event as method parameter
* Only classes are supported exception for method producers which can return a `ParameterizedType` (example `List<String>`) but injections must exactly match this type and `List`/`Set` injections are handled by looking up all beans matching the parameter