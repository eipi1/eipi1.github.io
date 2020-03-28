---
title: "Rust Into Implementation"
categories: [rust]
tags: [Impl Into]
---
#### TLDR
To do type annotations during method call for Impl Into try `Into::<dest_trait>::into(arg)`.
#### Scenario
I've to make HTTP request to some service and the request needs to serializable. To make actual request I'll use [http crate].
#### Let's get on with it
I've a enum of HTTP methods, let's say MyMethod. Why new enum when there's already a [struct Method]  in http crate; cause I need it to be serializable. Also the purpose is to tryout Into/From.
``` rust
#[derive(Debug, Serialize, Deserialize)]
enum MyMethod {  
    GET,  
    POST,  
}
```
I'll use the enum to construct http::Request -
``` rust
let method = MyMethod::GET;
let request_fut = Request::builder()  
    .method(???)
    .uri(&some_url)  
    .body(())  
    .unwrap();
```
Now the `method(...)` doesn't want MyMethod as a parameter, it needs http::Method or String, obviously! So I need to convert MyMethod in to http::Method.
So my initial thought is *I saw somewhere some code were using __.into()__! That was cool, I'll do the same!*
#### impl Into
[https://doc.rust-lang.org/std/convert/trait.Into.html](https://doc.rust-lang.org/std/convert/trait.Into.html)
``` rust
impl Into<http::Method> for MyMethod {
    fn into(self) -> http::Method {
        match self {
            MyMethod::GET => http::Method::GET,
            MyMethod::POST => http::Method::POST
        }
    }
}
```
That's all, fairly simple, right!
Now let's build the request -
``` rust
let method = MyMethod::GET;
let request_fut = Request::builder()
    .method(method.into())
    .uri(&some_url)
    .body(())
    .unwrap();
```
And run `cargo check`. Why not build/run? Because build will take lot more time than [cargo check] and will check for all error. But in rust, if program compiles, it works. 
```
error [E0284]: type annotations needed: cannot resolve `<http::method::Method as std::convert::TryFrom<_>>::Error == _`
  --> src/main.rs:89:14
   |
89 |             .method(method.into())
   |              ^^^^^^

error: aborting due to previous error
```
Uh-oh!! It seems compiler can't resolve the type, maybe specifying the type will work. But I've no idea how to specify type during function call :(. So I'll go with the way I already know first, by defining new variable, just to test my assumption.
``` rust
let method = MyMethod::GET;
let http_method:http::Method = method.into();
let request_fut = Request::builder()
    .method(http_method)
    .uri(&some_url)
    .body(())
    .unwrap();
```
Yay! It works!!
But some cosmetic issue remains; I don't want to define a new variable every time I call `into()`!
I want to specify type annotation during method call.
After some googling - it seems type annotation can be done as `Into::<expected_generic_type>::into(obj)`
``` rust
let method = MyMethod::GET;
let request_fut = Request::builder()
    .method(Into::<http::Method>::into(method))
    .uri(&some_url)
    .body(())
    .unwrap();
```
#### Unnecessary notes
It was harder than my expectation to find the above solution; maybe because of my limited understanding of rust.
At first I came up with slightly alternative syntax -
``` rust
let method = MyMethod::GET;
let request_fut = Request::builder()
    .method(<MyMethod as Into<http::Method>>::into(method))
    .uri(&some_url)
    .body(())
    .unwrap();
```
This syntax is maybe more generic and specially useful to resolve trait ambiguity during function call. See [here] for details.

[http crate]: https://docs.rs/crate/http
[struct Method]: https://docs.rs/http/0.2.1/http/method/struct.Method.html
[cargo check]: https://doc.rust-lang.org/cargo/commands/cargo-check.html
[here]: https://doc.rust-lang.org/reference/expressions/call-expr.html#disambiguating-function-calls
