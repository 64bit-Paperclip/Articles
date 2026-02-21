
# Reducing Vtable Indirection for Hot Path Performance

I've been working on a project that requires polymorphism, but specifically *runtime* polymorphism. The distinction matters. Polymorphism and inheritance are core strengths of C++ for organizing complex systems. However, in "hot path" code, the mechanism used to achieve this flexibility can become a major bottleneck.

If you know your types at compile time, you can avoid vtables entirely. Use CRTP, templated functions, or if you're feeling masochistic, switch statements on type enums. The compiler resolves everything statically and you pay zero runtime cost.

Runtime polymorphism is different. You have a collection of objects whose types you don't know until the program is running, and you need to call methods on them without knowing what they are. C++'s interfaces and virtual functions are the standard solution, but the mechanism that makes them work is what creates the bottleneck.

To understand this, let us briefly examine how C++ implements virtual dispatch.


## How C++ Implements Virtual Dispatch

When you declare a function virtual, the compiler generates infrastructure to support runtime polymorphism. Each object with virtual functions contains a hidden pointer (the vptr) that points to a vtable (virtual function table) for that object's type. The vtable is an array of function pointers, one for each virtual function the class implements.

When you call a virtual function, the CPU must:

1.  Load the object's vptr from memory
2.  Use that pointer to access the vtable
3.  Index into the vtable to find the correct function pointer
4.  Jump to that address

This indirection happens on every virtual call, regardless of how predictable the actual types are at runtime.

## The Problem: Vtable Indirection and Branch Misprediction

To support inheritance, every object instance carries a hidden pointer (vptr) to a virtual table (vtable) containing function addresses. This causes two primary hardware failures:

**The Pointer-Chasing Tax**: The CPU cannot execute the function until it performs a serial dependency chain: load the object → load the vptr → load the vtable → load the function address. If the vtable isn't in the cache, the CPU stalls for hundreds of cycles.

**Failed Branch Prediction**: Because the destination of a virtual call is unknown until the pointers are resolved, the CPU cannot accurately pre-fill the instruction pipeline. This leads to pipeline flushes, where the hardware must discard speculatively executed work and restart once the address is finally known.

## Why This Matters in Hot Paths

Hot path code executes millions or billions of times. A single virtual call might add only 10-20 nanoseconds of latency, but when that call sits inside a tight loop processing thousands of entities per frame, the overhead compounds catastrophically.

Modern CPUs are designed to execute hundreds of instructions in parallel through pipelining and speculative execution. Virtual dispatch breaks both of these optimizations. The pointer indirection creates serial dependencies that prevent instruction-level parallelism, and the unpredictable branch destination forces the CPU to guess, often incorrectly.

In performance-critical code (game engines, trading systems, physics simulations) these penalties can consume considerable amounts of execution time. The actual work being done is fast; the overhead of _deciding which work to do_ becomes the bottleneck.

## The Solution: Polymorphism Without Inheritance

The key insight is this: you don't need inheritance to achieve polymorphic behavior. What you need is a way to dispatch different implementations based on runtime state, and you can do that by embedding the function pointer directly in your data structure.

### Removing The Indirection

Consider the traditional approach with inheritance:
```cpp
class Shape {
public:
    virtual float area() const = 0;
};

class Circle : public Shape {
    float radius;
public:
    float area() const override { return 3.14159f * radius * radius; }
};

std::vector<Shape*> shapes;  // Array of pointers to shapes
```

Every call to `shapes[i]->area()` requires:

1.  Load the pointer from the vector
2.  Dereference to get the object
3.  Load the vptr from the object
4.  Load the vtable entry
5.  Jump to the function

Now consider storing the function pointer directly in the structure:

```cpp
typedef float (*AreaFunc)(float, float);

struct Shape {
    AreaFunc area;
    float param1;
    float param2;
};

float circle_area(float radius, float unused) {
    return 3.14159f * radius * radius;
}

float rect_area(float width, float height) {
    return width * height;
}

std::vector<Shape> shapes;  // Contiguous array of structures
shapes.push_back({circle_area, 5.0f, 0.0f});
shapes.push_back({rect_area, 3.0f, 4.0f});

for (const auto& shape : shapes) {
    float result = shape.area(shape.param1, shape.param2);
}
```

Now the call to `shapes[i].area(...)` requires:

1.  Load the Shape structure (which is contiguous in the vector)
2.  Load the function pointer (which is right there in the structure)
3.  Jump to the function

You've eliminated an entire level of indirection. The function pointer lives in the same cache line as the data you're already accessing. The CPU loads it all at once.

### Why This Is Faster

With traditional virtual dispatch, each object is allocated separately and the vector stores pointers. This means:

-   Poor cache locality (objects scattered in memory)
-   Two indirections (pointer to object, then vptr to vtable)
-   Unpredictable memory access patterns

With embedded function pointers:

-   Perfect cache locality (contiguous array)
-   One indirection (the function pointer itself)
-   Predictable linear memory access

The branch predictor can also learn patterns better because the function pointer is loaded from a predictable location. If you're processing a batch of circles followed by a batch of rectangles, the CPU will correctly predict the function pointer value and speculatively execute the right code path.


## When to Use This Approach

This technique shines when you're processing large collections of polymorphic objects in tight loops. The pattern to look for: iterating over hundreds or thousands of objects per frame where each iteration makes polymorphic calls.

Game engines updating entity components, physics simulations stepping through collision shapes, particle systems updating behavior, trading systems processing events; these are the use cases where cache locality and eliminating indirection actually matter.

If you're calling a polymorphic method once or twice per frame, or on a handful of objects, the overhead is negligible. The traditional virtual function approach is simpler and you should use it.

## A Note on Benchmarks

I haven't included performance comparisons because any numbers I provide would be highly contrived. The benefits depend entirely on your specific situation: how scattered your objects are in memory, whether types are batched or interleaved in your collections, your cache line size, how tight your loop is, what else is competing for cache.

The question isn't "is this always faster" but "does this solve a problem I have." If you're iterating over large polymorphic collections in hot paths and performance matters, try it and measure. If you're not, the added complexity isn't worth it.
