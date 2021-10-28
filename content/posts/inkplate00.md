---
title: InkPlate 10 dashboard
date: 2021-10-28
draft: false
description: "How to build an informative dashboard using InkPlate and Rust"
tags: [ "rust", "inkplate" ]
cover:
  image: /images/inkplate/inkplate01.png
  alternate: InkPlate 10 dashboard
---

# Intro

Recently there seem to be a resurgence of eink-based dashboards. E-Ink has many interesting capabilities: it does not consume energy to keep an image on screen and does not blast light into your eyes. It's also arguably prettier than a standard screen. 

There are many projects that use e-ink for mostly static content. In this post series we'll use the excuse of building a home dashboard to write us some Rust. For our goal we will use the super cool - and relatively cheap - [InkPlate](https://inkplate.io/).

The final result is like this (minus the Halloween decorations):

![](/images/inkplate/inkplate00.png#center)

We'll be cramming in few pixels:

1. A calendar (next few days).
2. A weather forecast.
3. The room temperature and air quality.
4. Some inkplate info (battery charge and so on).

# Forewarning

We will not program the InkPlate in Rust directly. While possible, it's still more hassle that it's worth if all the inkplate has to do is to display an image. InkPlate supports Arduino out of the box so we will use it to display the image. What we will do in Rust, is to create a web server that collects the data and composes the image. As we will see, it's easier that expected! 

# Basic HTTP Server in Rust

There are a lot of frameworks in Rust that help you create an HTTP server. I usually go with Hyper. While it's advertised as "low level" it's not opinionated and it's still reasonably easy to use. 

For example, routing in Hyper is as simple as:

```rust
match (req.method(), req.uri().path()) { 
	(&Method::GET, "/") | (&Method::GET, "/image.bmp") => {
		// handle root and /image.bmp
	},
	(&Method::GET, "/sleep") => {
	  	// handle /sleep 
	},
	(&Method::POST, _) => {
	  	// handle post
	},
	_ => {
		// return 404
	}
}
```

Of course since Hyper is super flexible you can pattern match with other things as well. Do you want to change the response based on the time of the day? Just add it to the `match` part! Rust will make sure you don't forget to cover any possible combination. 

## Hyper's only hard part: `make_service_fn`

People generally shy away from Hyper because the [`serve`](https://docs.rs/hyper/0.14.14/hyper/server/struct.Builder.html#method.serve) function has a super scary signature: 

![](/images/inkplate/hyper00.png)

Luckily we can borrow from the example above to get a minimally viable solution. We will build upon it in a bit:

```rust
let addr = "0.0.0.0:5527".parse()?; 
let make_service = make_service_fn(|_| async move { 
    Ok::<_, hyper::Error>(service_fn(move |req: Request<Body>| async {
        Response::builder() // we just answer OK without body for now
            .status(StatusCode::OK) 
            .body(Body::empty())
    }))
});
let server = Server::bind(&addr).serve(make_service); 
```

If you run it and point your browser to http://localhost:5277 you will see a beautiful blank page. 

## Globals

Our skeletal server is completely functional. It can, for example, inspect the received `Request` and produce an appropriate response. This is pretty limited though: for our purposes we need some way to:

1. Have a read-only repository

---

Happy Coding,

**Francesco Cogno**

