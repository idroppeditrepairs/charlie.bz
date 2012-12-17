# Why do singleton methods make Ruby slow?

Did you know that defining singleton methods on an object can make common operations (eg. arithmetic, hash/array indexing, method calls like `empty?`, `size`, etc) significantly slower?

If you don't believe me, run this script on your machine and take note of the somewhat surprising results:

    \ruby
    require "benchmark"
    GC.disable
    
    eval "def run; 10_000.times { #{"$a[5]\n" * 10_000} } end"
    
    $a = [1,2,3,4,5,6,7,8,9,10]
    
    puts "before singleton method definition"
    puts Benchmark.measure { run }
    
    def $a.x; end
    
    puts "after singleton method definition"
    puts Benchmark.measure { run }

On my machine, I get the following timings:

    \text
    before singleton method definition
      2.620000   0.000000   2.620000 (  2.622593)
    after singleton method definition
      5.820000   0.000000   5.820000 (  5.826397)

So what gives? Why would defining a completely unrelated singleton method on an array suddenly make indexing it more than twice as slow? To find out, we're going to have to learn a few things about Ruby's virtual machine and object system.

## The Ruby VM

MRI Ruby 1.9 and up uses a bytecode virtual machine internally. Before any Ruby code is executed, it is compiled into instructions for Ruby's virtual machine. Ruby allows us to peek behind the curtain and inspect the results of the compilation stage with the `RubyVM::InstructionSequence` class.

For example, take this simple looking line of Ruby:

    \ruby
    puts "Hello world".upcase

Let's compile it:

    \ruby
    puts RubyVM::InstructionSequence.compile(%q{
      puts "Hello world".upcase
    }).disasm

Using ruby-2.0.0-preview2, I get a bytecode listing that looks like this: (don't worry if yours is a little different)

    \text
    == disasm: <RubyVM::InstructionSequence:<compiled>@<compiled>>==========
    0000 trace            1                                               (   2)
    0002 putself          
    0003 putstring        "Hello world"
    0005 opt_send_simple  <callinfo!mid:upcase, argc:0, ARGS_SKIP>
    0007 send             <callinfo!mid:puts, argc:1, FCALL|ARGS_SKIP>
    0009 leave

Let's take a look at what's going on here. Ruby's VM is *stack based*, meaning that most instructions will pop their input values off the stack, do some computation, and the push their result values back on to the stack.

The `putself` instruction pushes the current value of `self` to the top of the stack. Then, the `putstring` instruction pushes its operand (`"Hello world"`) to the top of the stack. At this point, there are two objects on the stack - `"Hello world"` at the top and `self` in the next slot down.

`opt_send_simple` and `send` both do similar things. They pop a bunch of arguments and the receiver off the top of the stack, call the given method on the receiver with the collected arguments, and push the return value back on to the stack. It gets a little more complicated when it comes to splats, default arguments and unpacking return values, but we don't have to worry about that for now.

At `0005`, the `opt_send_simple` instruction pops `"Hello world"` off the stack, calls the `upcase` method on it and pushes the return value of `String#upcase`. The stack now contains `"HELLO WORLD"` at the top, and `self` is still waiting patiently in the next slot down.

The `send` instruction at `0007` pops 1 argument off the stack and then pops the receiver off the stack. The stack is now empty. It calls the `puts` method on the receiver (`self`), with `"Hello world"` as the argument, and pushes the return value (which happens to be `nil`).

Finally, the `leave` instruction pops the return value off the top of stack and exits the method.

## Speeding up common method calls

Everything in Ruby is a method call - even attribute access and arithmetic. When you call a method on an object, Ruby must look for that method in the object's class's method table. If the method is not found, Ruby tries to look for the method in each superclass until it reaches the root of the inheritance hierarchy - `BasicObject`. This is very slow.

Ruby uses a few techniques to speed up this otherwise slow process. One of these techniques involves using specialized instructions for common method calls on core classes. At time of writing, these optimized methods are `+`, `-`, `*`, `/`, `%`, `==`, `!=`, `<`, `<=`, `>`, `>=`, `<<`, `[]`, `[]=`, `length`, `size`, `empty?`, `succ`, and `=~`.

Here's the implementation of the `opt_aref` instruction - the optimized version of the `[]` method:

    \c
    DEFINE_INSN
    opt_aref
    (CALL_INFO ci)
    (VALUE recv, VALUE obj)
    (VALUE val)
    {
        if (!SPECIAL_CONST_P(recv)) {
            if (HEAP_CLASS_OF(recv) == rb_cArray
                    && BASIC_OP_UNREDEFINED_P(BOP_AREF, ARRAY_REDEFINED_OP_FLAG)
                    && FIXNUM_P(obj)) {
                val = rb_ary_entry(recv, FIX2LONG(obj));
            }
            else if (HEAP_CLASS_OF(recv) == rb_cHash
                    && BASIC_OP_UNREDEFINED_P(BOP_AREF, HASH_REDEFINED_OP_FLAG)) {
                val = rb_hash_aref(recv, obj);
            }
            else {
                goto INSN_LABEL(normal_dispatch);
            }
        }
        else {
          INSN_LABEL(normal_dispatch):
            PUSH(recv);
            PUSH(obj);
            CALL_SIMPLE_METHOD(recv);
        }
    }

This specialized instruction contains fast paths that do not perform any method lookups for `Array#[]` and `Hash#[]`. If `#[]` is called on an object that isn't an `Array` or `Hash`, the instruction falls back to a regular method call involving the slow method lookup we talked about earlier.

Additionally, if these optimized methods are ever redefined on any core class, Ruby will no longer use the fast path for that particular class and method - all subsequent calls will take the slow method call.

## Singleton methods and eigenclasses

In Ruby, when you define a singleton method on an object, you are actually defining an instance method on that object's eigenclass. Every object in Ruby is an instance of its own eigenclass - a subclass of the object's apparent class. These eigenclasses are created lazily by Ruby when they are needed.

    \ruby
    a = "I have an eigenclass"
    def a.foo; end
    def a.bar; end
    
    b = "I do not"

On line 2 of the code snippet above, a singleton method `foo` is defined on the object referenced by the `a` variable. At this point, Ruby creates a new subclass of `String` on the fly and rewrites the object's internal `klass` pointer to point to this newly created class. Note that even though the internal `klass` pointer has changed, the eigenclass does not override the `#class` method, so this is invisible from the perspective of your running Ruby code.

On line 3, another singleton method called `bar` is defined on `a`. This time, Ruby reuses the already existing eigenclass.

The object referenced by the `b` variable never has its eigenclass referenced, so Ruby never bothers to create it.

## Putting the pieces together

Ruby's laziness in creating eigenclasses is the key to understanding the cause of the strange slow down we observed above after defining a singleton method.

The type checks used by the optimized instructions looks like this uses the `HEAP_CLASS_OF` macro to find the class of an object. This macro has a very simple definition:

    \c
    #define HEAP_CLASS_OF(obj) (RBASIC(obj)->klass)

`RBASIC` is another macro that returns a pointer to the `RBasic` struct behind `obj`. The `RBasic*` is then dereferenced to find the class of `obj`.

If `obj`'s eigenclass has been brought into existence by defining a singleton method, then its internal `klass` pointer will have been rewritten to point to this eigenclass instead of the object's apparent class and it will no longer meet the criteria for taking the fast path in optimized method calls.

Note that the behaviour described post is entirely MRI specific. JRuby and Rubinius both perform more advanced optimizations and are able to avoid this little problem.