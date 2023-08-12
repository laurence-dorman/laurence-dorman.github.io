---
layout: post
title: Mandelbrot Explorer with Multithreading 🎇📈
description: A multithreaded mandelbrot set explorer and animation generator | C++ / SFML
date: 2021-05-17
---

>**Disclaimer: This is university coursework. The code is not intended to be used for any purpose other than to demonstrate the algorithms.**

## Table of contents
{:.no_toc}
* TOC
{:toc}

&nbsp;

***

&nbsp;

## Video Demonstration
<iframe width="560" height="315" src="https://www.youtube.com/embed/VWcVVKOz7Hg" title="YouTube video player" frameborder="1" allowfullscreen></iframe>

&nbsp;

***

&nbsp;

## Purpose

The purpose of the application is:
- To generate Mandelbrot sets as quickly and accurately as possible.
- To be able to navigate the Mandelbrot set easily.
- To both render the fractal instantly to screen and to output the generated images to disk for creating animations.
- The problem is that Mandelbrot sets take a long time to compute:
- My application attempts to solve this problem with multithreading and the use of a task-based parallelised system.

&nbsp;

***

&nbsp;

## Conclusion

> The full source code for this project can be found on [GitHub](https://github.com/laurence-dorman/Mandelbrot-SFML).