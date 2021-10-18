---
title: Let's build a Prometheus exporter in Rust
date: 2019-06-19
draft: false
description: A step-by-step tutorial on how to create a functioning Prometheus exporter in Rust
tags: [ "rust", "prometheus", "exporter" ]
cover:
  image: /images/lets-build-a-prometheus-exporter-in-rust/cover.png
  alternate: something good
---

## Intro

In this post we will build a very simple Prometheus exporter in Rust. [Prometheus](https://prometheus.io/) is a time-series database especially useful in storing and retrieving OS vital signs. It can be paired with [Grafana](https://grafana.com/) in order to create beautiful dashboards, like this one below: 

![](/images/lets-build-a-prometheus-exporter-in-rust/00.png)

Prometheus is peculiar because instead of *receiving* the events to store it goes on and retrieves them itself. There is no magic though: Prometheus just calls a preconfigured URI and expects a very specific plain text output. This is very elegant because this architecture decouples the service being monitored and the monitor adding an *exporter* service in between. It's the exporter's job to convert the service-specific metrics in a format Prometheus can understand and store. 

There are tons of pre-made exporters allowing you to monitor the server CPU, the DHCP, etc... with ease. But since this is dev post we won't stop at *combining* tools written by others. Instead, we will build an exporter of our own. This will be a very simple exporter but I wanted to show you how easy it is to do it so hopefully you can implement an exporter on your own. 

## The goal

We want to keep an eye on the size of a folder. Prometheus can store the folder size every 60 seconds and Grafana can plot the size over time beautifully. It also allows us to create alerts: we can, for example, be notified via Telegram if the folder grows beyond a threshold. 
But how we create the website needed by Prometheus in order to store the folder size? Enter Rust and an helper crate: [Prometheus exporter base](https://github.com/MindFlavor/prometheus_exporter_base). Also, as a bonus, being Rust we will be sure the memory/CPU footprint will be low.

### Prometheus exporter base

This crate is open source and MIT licensed (so you can use it freely) and it's designed to help you create a Prometheus exporter. It will handle most of the boilerplate required by Prometheus (such as rejecting anything but `GET` verbs and only answering to the `/metrics` URL). It also provides methods to format the output properly. So first thing first we need to import it by adding the relevant entry to the `[Dependencies]` section of out `Cargo.toml` file. This post is written using the version `0.3.0` of the crate so if you end up using a newer version you might have to account for breaking changes (if any).

The documentation is very terse: [https://docs.rs/prometheus_exporter_base/0.3.0/prometheus_exporter_base/](https://docs.rs/prometheus_exporter_base/0.3.0/prometheus_exporter_base/) but don't fret: all we have to do is to call the [`render_prometheus`](https://docs.rs/prometheus_exporter_base/0.3.0/prometheus_exporter_base/fn.render_prometheus.html) method. 

Its signature is this one: 

![](/images/lets-build-a-prometheus-exporter-in-rust/01.png)

What a mouthful! Basically we need to pass a closure that will be called at every GET request. The closure should return a String or and error.
Yes, Rust type system can get carried away. Let's break down the methods one by one.

#### Bind address

```rust
addr: &SocketAddr, 
```

This is the address our exporter will be listening to. We can pass 0.0.0.0 with a port of our choosing. For example this code will do:


```rust
let addr = ([0, 0, 0, 0], 32221).into();
```

#### Options

```rust
options: O
where
    O: Debug + Clone + Send + Sync + 'static
```

The `options` can be anything and it will be passed back to our closure at every call. The `O` type must also be cloneable, debuggable (meaning it must be printable in debug mode) and also must be sendable between threads. It also has to last forever (the `'static` lifetime). We do not need options so we create an empty type just for that. Notice how the `derive` trick makes it trivial:

```rust
#[derive(Debug, Clone)]
struct MyOptions {}
```

#### Closure

```rust
perform_request: P
where
    P: FnOnce(Request<Body>, &Arc<O>) -> Box<dyn Future<Item = String, Error = Error> + Send + 'static> + Send + Clone + 'static, 
```	    

This is a bit more complicated. What it means we must pass a function that takes the `http Request` as parameter. The second paramerer is the aforementioned custom option struct `O`, wrapped in an `Arc` (`Arc` allows multiple references of the underlying struct to be owned at the same time). The function must returns a `Future`: it must either resolve into a `String` - in case of success - or a `failure::Error` - in case something goes south. The bunch of other traits are generally less important besides the `'static` lifetime. The `'static` lifetime here warrants a mention: what it does is to restrict anything *captured* by the closure to live forever. In practice this just means that we either do not capture anything (easier) or make sure to move ownership into the closure.  

### Our exporter

Armed with this knowledge we can start creating a stub. Let's put this code as the `main` function of our exporter: 

```rust
fn main() {
    let addr = ([0, 0, 0, 0], 32221).into();
    println!("starting exporter on {}", addr);

    render_prometheus(&addr, MyOptions {}, |request, options| {
        Box::new({
            println!(
                "in our render_prometheus(request == {:?}, options == {:?})",
                request, options
            );

            ok("it's working!\n".to_owned())
        })
    });
}
```

It will compile and you will get a working webserver listening on port 32221 that:

1. Will only allow GET verbs.
1. Will only answer to the path `/metrics` as per Prometheus specification.
1. Will run our code once invoked. Right now, it will just print something in the exporter's console window and return a fixed string to the caller. 

Let's try it! After issuing `cargo run` in our crate we should be able to issue - in another terminal - `curl http://localhost:32221/metrics -v`. Since it's a standard HTTP GET you can use a browser to check it just the same. 

![](/images/lets-build-a-prometheus-exporter-in-rust/02.png)

Not bad for just few lines of crappy code!

## Folder size calculation

Our stub right now doesn't do anything useful. We want it to be able to calculate the size of a folder. Let's write a function for that: 

```rust
fn calculate_file_size(path: &str) -> Result<u64, std::io::Error> {
    let mut total_size: u64 = 0;
    for entry in read_dir(path)? {
        let p = entry?.path();
        if p.is_file() {
            total_size += p.metadata()?.len();
        }
    }

    Ok(total_size)
}
```

This function does not calculate the subfolders size but for our purposes will do just fine. Let's call it from our code. Start by adding this line to our `main` function: 

```rust
let future_log = done(calculate_file_size("/var/log")).from_err();
```

Note here the `done(...)` - `from_err()` dance that is common with future combinators. Now we replace the `ok("it's working\n".to_owned())` line with the future execution we just created: 

```rust
future_log.and_then(|total_size_log| {
    ok(format!("{}\n", total_size_log))
})
``` 

This is easy! Now if we run the exporter again we should receive the proper answer instead of a static string! Nice! Firefox screenshot below:

![](/images/lets-build-a-prometheus-exporter-in-rust/03.png)

Just for reference, now our code is like this:

```rust
#[derive(Debug, Clone)]
struct MyOptions {}

fn calculate_file_size(path: &str) -> Result<u64, std::io::Error> {
    let mut total_size: u64 = 0;
    for entry in read_dir(path)? {
        let p = entry?.path();
        if p.is_file() {
            total_size += p.metadata()?.len();
        }
    }

    Ok(total_size)
}

fn main() {
    let addr = ([0, 0, 0, 0], 32221).into();
    println!("starting exporter on {}", addr);

    render_prometheus(&addr, MyOptions {}, |request, options| {
        Box::new({
            println!(
                "in our render_prometheus(request == {:?}, options == {:?})",
                request, options
            );

            let future_log = done(calculate_file_size("/var/log")).from_err();
            future_log.and_then(|total_size_log| {
                ok(format!("{}\n", total_size_log))
            })
        })
    });
}
```

## Prometheus compliance

This is all well and good but it does not mean our output is Prometheus compliant. In order to do so we should follow a specific format. Luckily the above helper crate has some methods to help us in this endeavor too. 
We just need to create an instance of `PrometheusCounter`. The `new(...)` constructor requires:

1. A counter name. This is up to you to get correctly and I refer you the official Prometheus documentation for it. I will just use `folder_size` for this post.
1. A counter type. Again, please refer to the [Prometheus documentation](https://prometheus.io/docs/concepts/metric_types/) for this. I will go with `counter` on this one. 
1. A counter help text. This is entirely optional but might help other people to understand what your counter is meant to do. 

Once created we call the `render_header()` method so the crate will output the required header for our counter. The code will be like this:

```rust
let pc = PrometheusCounter::new("folder_size", "counter", "Size of the folder");
let mut s = pc.render_header();
```
That settles the header. All we need to do now is to output the values. Each counter can optionally have one or more attributes. For example our folder size counter can have the `path` attribute: this way you could have more than one instance of the same counter in a single response, each indicating a different resource. Our crate allows you to specify the attributes as slice of tuples: attribute-value. For our example we can use a vector like this: 

```rust
let mut attributes = Vec::new();
attributes.push(("path", "/var/log/"));
``` 

Now we can call the `render_counter` function passing the attributes and the value. Like this: 

```rust
pc.render_counter(Some(&attributes), total_size_log);
```

All we need to do is to append the rendered counter to the header we just obtained and we are done. Since we already have `s` that is a mutable `String` we can append (push) the correctly formatted `String` there:

```rust
s.push_str(&pc.render_counter(Some(&attributes), total_size_log));
```

## Result

The final code is like this:

```rust
use futures::future::{done, ok, Future};
use prometheus_exporter_base::{render_prometheus, PrometheusCounter};
use std::fs::read_dir;

#[derive(Debug, Clone)]
struct MyOptions {}

fn calculate_file_size(path: &str) -> Result<u64, std::io::Error> {
    let mut total_size: u64 = 0;
    for entry in read_dir(path)? {
	let p = entry?.path();
	if p.is_file() {
	    total_size += p.metadata()?.len();
	}
    }

    Ok(total_size)
}

fn main() {
    let addr = ([0, 0, 0, 0], 32221).into();
    println!("starting exporter on {}", addr);

    render_prometheus(&addr, MyOptions {}, |request, options| {
	Box::new({
	    println!(
		"in our render_prometheus(request == {:?}, options == {:?})",
		request, options
	    );

	    let future_log = done(calculate_file_size("/var/log")).from_err();
	    future_log.and_then(|total_size_log| {
		let pc = PrometheusCounter::new("folder_size", "counter", "Size of the folder");
		let mut s = pc.render_header();

		let mut attributes = Vec::new();
		attributes.push(("path", "/var/log/"));
		s.push_str(&pc.render_counter(Some(&attributes), total_size_log));

		ok(s)
	    })
	})
    });
}
```

And the result is like this one:

![](/images/lets-build-a-prometheus-exporter-in-rust/04.png)

## Conclusion

Once added to Prometheus we can create beautiful dashboards like this one: 

![](/images/lets-build-a-prometheus-exporter-in-rust/05.png)

PS: I have built a couple of exporters myself, so if you want to see a more real world examples please refer to [my GitHub profile](https://github.com/mindflavor)!

---

Happy Coding,

**Francesco**

