# Deadlocks in Rust

Rust is an awesome language which offers memory safety and safe sharing between threads. All this without a GC. But even then, a Rust program can deadlock. Let's have a look at the following:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    // This is a channel, for sender to signal reciever that it sees flag to be true.
    let (tx, mut rx) = oneshot::channel::<i64>();

    let notify_send = Arc::new(Notify::new());
    let notify_recv = Arc::clone(&notify_send);

    let sender = tokio::spawn(async move {
        println!("Waiting to be notified ...");
        notify_recv.notified().await;
        println!("Sending value over channel");
        let _ = tx.send(100);
    });

    let reciever = tokio::spawn(async move {
        println!("Trying to recieve value ...");
        if let Ok(data) = rx.try_recv() {
            println!("Recieved {} over channel", data);
            println!("Notifying other ...");
            notify_send.notify_one();
        }
    });

    // I am pretty sure you can see why a deadlock occurs.

    sender.await?;
    reciever.await?;
    Ok(())
}
```
The `sender` task pauses (note: it won't block because of the `await`) on `notify_recv.notified()`. The execution can resume only when `notify_send.notify_one()` is executed in the `receiver` task. But `receiver` task will execute this only when `rx.try_recv()` completes, which it won't because `tx.send(10)` never executes. *This program will deadlock*.

## What do we do?
**We build a tool!**

Rust has an awesome repo of libraries, called [crates.io](https://crates.io). One library is called `Syn`, which is a parser for Rust code, ie, it generates an AST from Rust code. Although it was created for use in proc-macros, who cares? We can use it to build a *static-analyzer*, for deadlocks.