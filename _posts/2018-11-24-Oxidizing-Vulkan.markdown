---
layout: post
title:  "oxidize!(Vulkan)"
date:   2018-11-24 22:26:48 +0100
categories: vulkan, rust
---

> Update1: clarifications, updated LunarG structure

---

Let's say you don't like using somebody's else Crates and you have enough time to rustify Vulkan on your own. Here I will show you how to not shoot yourself in the foot when calling Vulkan from `libvulkan.so` directly rather than some wrappers.

Agenda:
1. (optional) Extract libvulkan.so from official LunarG SDK
2. Link Rust binary against it
3. Prepare rustified version of function(s) declarations
4. Call it in unsafe blocks

# Extracting vulkan shared object

This step is optional because you most probably have `libvulkan.so` already installed on your system - either by your OS (assuming Linux) or gpu drivers.

For the sake of completness we are gone prepare eveything from scratch. Head to the [LunarG](https://vulkan.lunarg.com/) website and download SDK bundle. After extracting head to `x86_64/lib/` directory. We are intersted in 1 file - vulkan loader: **libvulkan.so.1.2.131**.

Copy them to your Rust project directory (prefferably in some nested folder to not clutter main one). Let's also prepare symlink just for ease of use (similar to what LunargG prepare for us):

```bash
ln -s libvulkan.so.1.2.131 libvulkan.so
```

# Linking against loader

Now that proper lib is inside our Rust project we can inform our crate about it's existance. Create `build.rs` file (if don't have one) and add linker search path + library we want to link against:
```rust
fn main()
{
    println!( "cargo:rustc-link-search=./path/to/loader" );
    println!( "cargo:rustc-link-lib=vulkan" );
}
```

Path to loader is where you pasted loader and created symlink.

# Rusty version of C API functions

This is probably the biggest hassle. Given `vulkan.h` which contains all C API interface we need to **oxidize** function signature to let rustc understand it. For the sake of simplicity we will use `vkEnumerateInstanceVersion` as this is only taking one uint argument and returning `VkResult`.

Let's see it in code:
```rust
pub type VkResult = i32;

extern "C" 
{
    pub fn vkEnumerateInstanceVersion(pApiVersion: *mut u32) -> VkResult;
}
```


