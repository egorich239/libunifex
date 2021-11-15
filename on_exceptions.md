**Disclaimer** This argument is NOT part of unifex mainline, all views are mine, github.com/egorich239

*tl;dr;* C++ sender/receiver model (e.g. unifex) should be unopinionated w.r.t. exceptions. 
In particular it should not require `set_error(std::exception_ptr)` from receivers.

Note: I will use _unifex_ as a shorthand for _sender/receiver model proposed in P2300_ in this document.

Longer version:

1. Exceptions in C++ callee/caller propagate from the callee to the caller.
Exceptions in C++ coroutines propagate from coroutine to its continuation. In sender/receiver terms
both models propagate from sender to receiver. While unifex _could_ technically allow throwing an
exception in receiver's `set_value()` body and catching it in the upstream sender, it absolutely must
not do it; entering `set_value()` must insulate the sender from the exceptions thrown by the receiver.

2. There is no good way to handle a potential exception thrown by `val.~T()` after `unifex::set_value(rec, val);` returned.

3. Exceptions are _a_ mechanism to propagate information about an error alongside a _particular kind_ of
sender/receiver chain, namely synchronous call stack. It is not the only one, neither is it the default one
for the C++ community. Large parts of the industry (embedded, game developers, search engines) are not 
willing to pay the price of exceptions+rtti in their binaries, not now or ever.

4. Unifex is glue code, it does not contain business logic of an application. It must not impose restrictions
on the users of the library, unless these restrictions are absolutely necessary for the library to work.
With the above caveat of `val.~T()` it is possible to enforce non-throwing semantics of passing a value from
sender to receiver. Once done, there is no technical reason to expect `std::exception_ptr` from the glue code.
This implies that if the sender does not `set_error(exc_ptr);`, the receiver will never receive an exception ptr.

5. The core of unifex sender/receiver model can be implemented in circa C++11. It integrates well with such
power features as coroutines, but it does not require them. When combined with (4), it forms an interesting
value proposition for those who write their C++ code without exceptions.

## 1. Exceptions should not be propagated from receiver to sender.

Consider the following implementation of a trivial `just_void` sender. Its only duty is to send `(void)` value
signal to the receiver.

```c++

template <typename R>
struct just_void_op {
	void start() noexcept & { unifex::set_value((R&&) r); }
	R r;
};

struct just_void {
	using value_types = overloads<void()>;
	using error_types = overloads<>;

	template <typename R>
	auto connect(R&& r) noexcept(std::is_nothrow_moveable_v<R>) {
		return just_void_op{(R&&) r};
	}
};

```

Consider now the following two receivers, and the trigger function.

```c++

template <typename NextR>
struct propagate_and_throw {
	void set_value() {
		unifex::set_value((NextR&&) *next);
		throw 42;
	}
	NextR* next;
};

struct final_receiver {
	using op_t = just_void_op<propagate_and_throw<final_receiver>>;

	final_receiver() 
		: op(unifex::connect(
			just_void{}, 
			propagate_and_throw<final_receiver>{this})  {
	}

	void set_value() { op.~op_t(); }
	void start() { unifex::start(op); }

	union { op_t op; }
};

void trigger() {
	final_receiver r;
	r.start();
}

```

If `unifex::set_value()` does not insulate the exception and it pops to `just_void_op::start()`.
How can `just_void_op::start()` handle this situation? Well, `start()` could catch the exception. 
But it cannot do _anything_ to it: in fact with the `final_receiver` catch block will execute with
an already destroyed state of `just_void_op`! The requirement that the receiver does not ever throw
after the sender's operation state is destroyed should be obeyed but, is barely enforcable by the 
C++ language. Furthermore, the sender has a choice of `set_value()`, `set_error()` or `set_done()` 
on the receiver again or `std::terminate()`. ALL of it can be done by the receiver itself in a safer
way, since the receiver has a much better visibility into its own state than the sender, there is no
need to for the roundtrip to sender.

Finally, propagating exceptions from the receiver to the sender would be the only precedent of error
propagation in _that_ direction, all native sender/receiver capabilities (i.e. synchronous function
calls and coroutines) never propagate errors from the receiver side to the sender side. In addition
exceptions would be _the only_ kind of errors that would support propagation in that direction, and
only because unifex sender/receiver model is implemented via subroutine calls. As a thought experiment:
if unifex hypothetically implemented passing from sender to receiver via spawning a new process, 
it would not be possible to implement the same kind of exception propagation from sender to receiver.

## 2. The problem with `set_value()` argument destruction. 

In the previous section we figured that _a receiver should not throw after it has (potentially) caused
operation state destruction_ (such destruction might be caused by the receiver directly, or indirectly
via a `unifex::set_*()` call to the next receiver). While transferring control from the sender's
evaluation of `set_value()` arguments to the first statement of the receiver's `set_value()` can be very
closely mapped to the mechanics of the `return` statement, there is one important difference: the object
`v` of the `return v;` statement is destroyed in the receiver's frame (i.e. the caller) whereas object
`v` of `unifex::set_value(rec, v);` statement is destroyed in the sender's frame.

Such a `v.~T()` destructor call might happen _after_ the destruction of the operation state, and might
generally throw. There is no good way to handle such an exception, `std::terminate()` seems the only
feasible option.

Another option is however to ensure that `v.~T()` never throws either because it is a value type that
has `noexcept`, or because `v` is in fact an reference to an non-temporary object.

## 3. Exceptions are neither the only nor the default option for C++ error handling.

The claim of this section is that the *utility of a standard C++ library that does not
require exception support is strictly higher than otherwise*, i.e. it will be used by
a larger number of C++ engineers, teams and projects.

Exceptions are _a_ mechanism to _implicitly_ propagate information about an error alongside a 
sender/receiver chain of nested synchrnous function calls. They require RTTI for their work, in other
words they require keeping mangled names of large portions of types in a program (or shared library).
They also require exception handlers table, and/or additonal code to handle the failure scenario.
Many consider it a high price to pay, and use alternative error handling strategy.

One alternative design is executed in `folly::Expected<T, E>` or `absl::StatusOr<T>`. They achieve very
similar goals for the price of explicit error handling. With the help of convenience macros the mental
overhead of manual error handling is low enough and gets close to `let x = subcall()?;` statement in Rust.

There is also `std::error_type`, and plain old `int` error messaging style. The most evil side of the `int`
error handling, does not manifest itself in sender/receiver. While it is possible and tempting to
ignore the return value of a system call, it is much harder to do so if this value is published into
`set_error()` channel by a sender.

Exceptions will never be accepted by the whole C++ community. Every CppCon has at least one presentation 
from the perspective of an embedded, automotive, game or Mars rover developers - all of them build with
`-fno-exceptions` (sidenote: and oftentimes without malloc). The embedded C++ subgroup points out that 
anything related to exceptions is out of scope for embedded C++ standard (AI: I am bluffing here, I don't
even know the right name of that subgroup). This attitude is not specific to embedded developers: the
largest search engine in the world is built without exceptions.

## 4. Unifex should be non-opinionated, if technically feasible.

This is direct implication of the previous section. Unifex is a glue code for the C++ engineer to do its
job. The less assumptions it does about engineer's environment, the wider it will be adopted. In particular,
if unifex does not make assumptions about presence of exceptions, related types, and users' attitude
toward exceptions, it will be accepted wider than otherwise.

In the next section I argue, that unifex can be structured in such a way that the library itself does not
throw while passing signals (whether values, or errors) from sender to its receiver.

There is an example in C++ of an unopinionated (w.r.t. exceptions) implementation of sender/receiver model.
It is synchronous nested function calls. Their mechanics supports exceptions but does not require
their presence.

## 5. Unifex glue can be implemented `noexcept`.

`unifex::set_value(rec, s1, ..., sN)` always sits between sender and receiver's `set_value(r1, ..., rN)`.
I use different letters to designate that a type conversion `typeof(sK) -> typeof(rK)` happens in between.

I think we should have two policies:

1. If we require that all such implicit conversions from `s1, ..., sN` to `r1, ... rN` are non-throwing.

2. We can also either require that destructors of `r1.~R1(), ..., rN.~RN()` are non-throwing, or guard them
with `std::terminate()`. In the section 2 above I argue that there is no good way to handle throwing 
destructors - both sender and receiver have fully finished their execution by that moment, and the operation 
state might even be already destroyed.

There is also an option to enforce passing references to non-temporary objects (stored in operation state)
instead of values, but I don't really like it. It narrows down policy (1) above to a particular kind of
implicit conversions (too narrow, I think; e.g. `short -> int` is non-throwing and is always possible
whereas `short& -> int&` - not). It also does not save from the problem of temporaries destruction, but
simply moves it to another point during the execution - the op-state destruction. And there is no good
way to handle throwing destructors during op-state destruction either.
