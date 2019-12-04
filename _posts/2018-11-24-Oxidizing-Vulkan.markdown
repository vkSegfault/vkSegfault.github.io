---
layout: post
title:  "Oxidizing Vulkan!"
date:   2018-11-24 22:26:48 +0100
categories: jekyll update
---

Let's say you don't like using somebody's else Crates and you have enough time to rustify Vulkan on your own. Here I will show you how to not shoot yourself in the foot when calling Vulkan from `libvulkan.so` directly rather than some wrappers.

Agenda:
1. (optional) Extract libvulkan.so from official LunarG SDK
2. Link Rust binary against it
3. Prepare rustified version of function(s) declarations
4. Call it in unsafe blocks

# Extracting vulkan shared object

This step is optional because you are most probably 
