---
title: SIMD usage in C++, C# and Rust
categories: [CPU, SIMD]
tags: [cpu, simd, cpp, csharp, rust]     # TAG names should always be lowercase
---

Everyone heard of SIMD, yet not so many use it. There are probably three main reasons for that: it's hard to write it correctly, it's not safe and, most important, compiler can write it for us in many cases (auto vectorization). So why even bother? And answer is: bacause compilers are not that smart and for many performance-critical paths we can't just rely on the will of compiler. You can easily break compiler-deduced vectorization by using `break` or `goto`, when number of iterations are not known at compile time, where there are dependencies between iterations or when number of elements is not aligned to your SIMD vector width. So let's defy the odds and write intrinsics explicitly. Below is the whirlwind tour of SIMD usage in 3 popular languages.

> Note that this blogpost is pure language usage comparasion, not benchmark. There is a lot of benchmarks out there so there is no need to repeat them.

> C++ code was tested on **GCC 11.1.0**, C# on **.NET 5.0.203** and Rust on **rustc 1.52.1**. Everything compiled on Linux with small exception to C# when easy way to verify JITted assembly was VS Code plugin that runs only on Windows.


## Standarization

C++ has no standarized SIMD usage at all, even for SSE instructions introduced 20 years ago, which are available for almost all x86_64 CPUs as of 2021. To use wide lanes we must provide compiler-specific headers and use all instructions directly. In case of GCC it all looks like this:
```cpp
#include <xmmintrin.h>   //for SSE
#include <emmintrin.h>   //SSE2
#include <pmmintrin.h>   //SSE3
#include <smmintrin.h>   //SSE4.1
#include <nmmintrin.h>   //SSE4.2
#include <immintrin.h>   //AVX, AVX2, AVX-512
```
All SSE includes give us `__m128` types while `immintrin.h` results in `__m256` and `__m512`. Although we have only this raw approach for now there is some work happening to standarize it in [future](https://en.cppreference.com/w/cpp/experimental/simd). It's worth mentioning that in the 90's ``valarray`` has been introduced.  Where vector and array are just templated containers without any mathematical meaning, vallarray was concept of array conataining numbers only, which can execute wide operations. Unfortunatelly development stalled and till today there is no exact matching to SIMD inctructions, so we must rely on auto vectorization.

At the other end of the spectrum there have been C# which had fully standarized SIMD usage ever since .NET Core 3.0. We don't have to bother about any intrinsics directly, everything is under nice high level constructs:
```c#
using System.Numerics;
```
From now on you can use types such a `Vector2`, `Vector3`, `Vector4` and `Matrix3x2`, `Matrix4x4`. There are even more specialized types like `Plane` or `Quaternion`. All those use single precission floating point numbers. If you want use doubles or ints and wider operations you can use `Vector<T>` type which also let you write hardware independent code.
> While .NET Core has SIMD out of the box, if you still use .NET Framework you need ***System.Numerics.Vectors*** NuGet package.

And there is Rust, the new kid on the block. Given it's pretty young (1.0 released in 2015), it should handle intrinsics in some meaningful way, right? Actually, the situation is  a bit more complicated. Since 2018 Edition (1.27 version to be precise) we can use SIMD basically directly (similar to C++), but at the same time it's all nicely wrapped into standard library:
```rust
use std::arch::x86_64::*;
```
Ever since SIMD arch module introduction there was always some talk about creating portable (rusty) way to achieve wide lane instructions directly without compromising Rust safety and syntax. I've checked 2 previous attempts: [simd](https://crates.io/crates/simd) and it's successor [packed simd](https://crates.io/crates/packed_simd), but they are outdated and none of them compile as of rustc 1.51. The latest is [Portable SIMD Project Group](https://blog.rust-lang.org/inside-rust/2020/09/29/Portable-SIMD-PG.html). Once (or rather *if*) they finish we should be able to use constructs like `f32x4` directly with all mathematical operations on them.

> For C++ and Rust this list of SIMD operations will definitely come in handy: [Intel Intrinsics Guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide)

## Safety

As you can blindly guess, C++ way of CPU intrinsics is not safe in any way. What will happen when you run instructions on hardware that does not support it is UB, and most probably segfault. There is a way though to ensure runtime safety, but once more not portable. It's called **CPUID** and here is an example:
```cpp
#include  <cpuid.h>

//(...)

int info[4];
__cpuid_count(0x0000'0001, 0, info[0], info[1], info[2], info[3]);
if ( info[2] & ((int)1 << 28) != 0 ) //check bit for AVX presence
{
	//do something with AVX
}
```
C# on the other hand does everything fully safe and potentially faster (!). At least theoretically (depends whether JITted code optimized for specific hardware gains more than CLR runtime workload[^1]). This code checks at runtime how wide (how many lanes) is the vector of SIMD data:
```c#
var lanes = Vector<int>.Count;
```
Of course, it implicitly matches to SSE (4 lanes), AVX (8 lanes), AVX-512 (16 lanes) or - in case of no SIMD inctructions detected - scalar (1 lanes). While C++ and Rust must compile AOT and assume what is typical hardware your program will run on, C# doesn't need to. CLR compiles it JIT dynamically checking on what CPU it's running. If it finds that we are using  CPU modern enough to support e.g.: AVX-512, why bother with something less wide? This is really elegant way to acomplish portable SIMD usage comparing to C++ and Rust which must struggle with compile time decisions.

Rust has some neat syntax to create different functions per every instruction family and then dynamically decide which one to run. First to hint **rustc** to compile some function with avx we need feature attribute:
```rust
#[target_feature(enable = "avx")]   //note that rustflags have precedence over this attribute
```
and we can prepare many other functions for every use case e.g.: SSE, AVX, AVX-512 and fallback scalar versions compiled right away. Then we can check at run time which features our hardware supports and run specific path based on it: 
```rust
if is_x86_feature_detected!( "avx" ) {
```
We can check (also) statically whether particular target feature is enabled 
```rust
#[cfg(target_feature = "avx")]
fn foo() {   //foo is only compiled when avx is enabled
#[cfg(not(target_feature = "avx"))]
fn foo() {   //this foo is only compiled when avx is not available

//or via macro:
if cfg!(target_feature = "avx") {
```
> Small tidbit here: above runtime check in Rust is actually made by the same solution presented already in C++ part: CPUID.

Please note that all fn's that use **std::arch** module must be marked as **unsafe**!

## Example
Here we have simple loop example for every language that sums 256 bits vector from a big array.

- C++

```cpp
__m256  v8temp;
union { __m256  v8sum; float  f8sum[8]; };
v8temp = v8sum = _mm256_setzero_ps();

std::array<float, 1'200'000> arr {};
arr.fill( 23.74 );

for( int  i = 0; i < arr.size(); i+=8 )
{
	v8temp = _mm256_set_ps( arr[i+7], arr[i+6], arr[i+5], arr[i+4], arr[i+3], arr[i+2], arr[i+1], arr[i+0] );
	v8sum = _mm256_add_ps( v8sum, v8temp );
}
```

- C#

```c#
var lanes = Vector<int>.Count;

float[] arr = new  float[1_200_000];
Array.Fill<float>( arr, 23.74f );
	
var v8sum = new Vector<float>();

for( int i = 0; i < arr.Length; i+=lanes )
{
	var v8temp = new Vector<float>( arr, i );
	v8sum += v8temp;
}
```

Note how inside if condition we don't operate on any specific size corresponding to SIMD lanes width. It's resolved at runtime and once it's written we don't need to worry about it (on my machine `v8sum` vector resolved to 8 elements).

- Rust

```rust
let (mut v8temp, mut v8sum) = ( _mm256_setzero_ps(), _mm256_setzero_ps() );
let arr: [f32; 1_200_000] = [ 23.74; 1_200_000 ];

//first version
for i in arr.iter().step_by(8) {
	v8temp = _mm256_loadu_ps( i );
	v8sum = _mm256_add_ps( v8sum, v8temp );
}
let v8sum_unpacked: (f32, f32, f32, f32, f32, f32, f32, f32) =  std::mem::transmute( v8sum );

//second version
arr.chunks(8).for_each(|chunk| {
	v8temp = _mm256_loadu_ps( &chunk[0] );
	v8sum = _mm256_add_ps( v8sum, v8temp );
} );
let v8sum_sliced = std::mem::transmute::< __m256, [f32; 8] >(v8sum);
```

Rust is pretty straightforward if you are already familiar with syntax, but there are few things to note here. 

Example shows 2 loops, one using iterators and one chunks. It was not clear at first but for faster computations we should go with functional style. ***For loop*** uses iterators which are indirection to data, this is pretty significant with high performance code. Chunks approach on the other hand operates on raw data, which results in faster compute times.

Another thing worth mentioning is **mem::transmute**. This fn is responsible for bitcasting and under the hood it's just union type.

And last but not least fact is that arrays in rust do not hold aligment requirements by default. That's why we need to use `_mm256_loadu_ps` instead of `_mm256_load_ps` (notice lack of `u`).  

Once **std::simd** stabilization finishes we can expect more rusty syntax:
```rust
let (mut v8temp, mut v8sum) = ( f32x8::splat(0.0), f32x8::splat(0.0) );
arr.chunks(8).for_each(|chunk| {
	v8temp = f32x8::from_array( chunk.try_into().expect("") );
	v8sum += v8temp;
});
```
Notice how natural using `v8sum += v8temp` will be, those are just pure SIMD constructs. If you want to check outgoing efforts to stabilize it right now, add this as dependecy to your project:
```bash
core_simd = { git = "https://github.com/rust-lang/stdsimd" }
```

### Compiler flags

One more thing worth mentioning are compiler flags. While CLR doesn't need anything, compilers for C++ and Rust must provide information what characteristics of CPU we want to activate. For GCC (unless it's SSE which can be compiled right away) we need to pass **-mavx** or **-march=native**. Rustc on the other hand needs specific rustflags in .cargo/config.toml:
```toml
[build]
rustflags = [ "-C", "target-feature=+avx,+avx2" ]
```
> **rustflags** have precedence over **attributes** !

## Bonus - veryfing assembly
As small bonus: let's see how to check if our binaries are actually using AVX instructions. In C++ and Rust it's pretty straightforward, we have already compiled blob so we can use well known tool called **objdump** (*readelf* or *nm* will also do, if that's what you prefer):
```bash
objdump -d simd_exec > simd_exec.asm
grep "ymm" simd_exec.asm
```
in my case output looks like this:
```asm
    a71e:	c5 fc 29 00          	vmovaps %ymm0,(%rax)
    ad08:	c5 fc 28 06          	vmovaps (%rsi),%ymm0
    ad0c:	c5 fc 58 02          	vaddps (%rdx),%ymm0,%ymm0
    ad10:	c5 fc 29 07          	vmovaps %ymm0,(%rdi)
```
Here we check for **ymm** presence in produced assembly; those are 256bit wide registers used for AVX and AVX2. We can also grep for specific instruction like **vaddps** which uses ymm registers to execute addition.

C# is bit harder to check. When we compile our code, we don't produce machine level assembly but rather MSIL (Microsoft Intermediate Language) that is later interpreted/compiled further at runtime. For SIMD produced MSIL has no use, we need machine instructions, which means we need to run our code on some specific processor. CLR will produce assembly that is no longer portable yet perfectly fitted for our hardware. If you are using Visual Studio Code there is really nice extension called [**.NET Insights**](https://github.com/jashook/vscode-dotnet-insights) where we can inspect our JITted code. Unfortunatelly as of writing this article it's not crossplatform and only works on Windows machine. Here is output of SIMD code used in C#:
```asm
G_M19891_IG04: ;; offset=0152H
000152  C4E34D18F701 vinsertf128 ymm6, ymm6, ymm7, 1
000158  C5CC58F0 vaddps ymm6, ymm6, ymm0
00015C  83C308  add  ebx, 8
00015F  81FB804F1200 cmp ebx, 0x124F80
000165  0F8CEFFEFFFF jl G_M19891_IG03
```


## Footnotes
[^1]: I imagine this may happen on modern CPUs that support AVX-512
