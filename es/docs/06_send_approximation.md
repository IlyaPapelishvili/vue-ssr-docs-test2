# `Send` aproximación

Algunas máquinas de estado `async fn` son seguras para enviarse a través de subprocesos, mientras que otras no. El hecho de si un `async fn` `Future` es `Send` se determina por si un tipo que no es `Send` se mantiene en un punto `.await` . El compilador hace todo lo posible para aproximarse cuando los valores pueden mantenerse en un punto `.await` , pero este análisis es demasiado conservador en varios lugares hoy en día.

Por ejemplo, considere un tipo simple que no es `Send` , quizás un tipo que contiene un `Rc` :

```rust
use std::rc::Rc;

#[derive(Default)]
struct NotSend(Rc<()>);
```

Las variables de tipo `NotSend` pueden aparecer brevemente como temporales en `async fn` s incluso cuando el tipo `Future` resultante devuelto por `async fn` debe ser `Send` :

```rust
async fn bar() {}
async fn foo() {
    NotSend::default();
    bar().await;
}

fn require_send(_: impl Send) {}

fn main() {
    require_send(foo());
}
```

Sin embargo, si cambiamos `foo` para almacenar `NotSend` en una variable, este ejemplo ya no compila:

```rust
async fn foo() {
    let x = NotSend::default();
    bar().await;
}
```

```
error[E0277]: `std::rc::Rc<()>` cannot be sent between threads safely
  --> src/main.rs:15:5
   |
15 |     require_send(foo());
   |     ^^^^^^^^^^^^ `std::rc::Rc<()>` cannot be sent between threads safely
   |
   = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::rc::Rc<()>`
   = note: required because it appears within the type `NotSend`
   = note: required because it appears within the type `{NotSend, impl std::future::Future, ()}`
   = note: required because it appears within the type `[static generator@src/main.rs:7:16: 10:2 {NotSend, impl std::future::Future, ()}]`
   = note: required because it appears within the type `std::future::GenFuture<[static generator@src/main.rs:7:16: 10:2 {NotSend, impl std::future::Future, ()}]>`
   = note: required because it appears within the type `impl std::future::Future`
   = note: required because it appears within the type `impl std::future::Future`
note: required by `require_send`
  --> src/main.rs:12:1
   |
12 | fn require_send(_: impl Send) {}
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
```

Este error es correcto Si almacenamos `x` en una variable, no se descartará hasta después de `.await` , momento en el cual el `async fn` puede estar ejecutándose en un hilo diferente. Dado que `Rc` no es `Send` , permitirle viajar a través de subprocesos sería poco sólido. Una solución simple a esto sería dejar `drop` el `Rc` antes de `.await` , pero desafortunadamente eso no funciona hoy.

Para solucionar este problema con éxito, es posible que deba introducir un ámbito de bloque que encapsule las variables que no sean de `Send` . Esto hace que sea más fácil para el compilador decir que estas variables no viven en un punto `.await` .

```rust
async fn foo() {
    {
        let x = NotSend::default();
    }
    bar().await;
}
```
