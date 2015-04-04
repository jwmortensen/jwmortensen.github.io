---
layout:  post
title: "Gaussian Processes" 
date: 2015-04-03 20:49:00
categories:
- gaussian processes
tags: featured
---

When dealing with nonlinear data a standard data analysis approach often includes techniques 
such as polynomial regression or natural splines. However, these may not always be the best
tool for the job. One example of a situation where these methods may not perform as well is 
when data is distributed spatially. In these types of situation one method that can be 
particularly useful is a Gaussian process.

### What is a Gaussian Process?

A Gaussian process is a form of stochastic, or random, process. We can think of 
stochastic processes as a potentially infinite collection of random variables over a domain
\\( \mathcal{T} \\). This domain could be temporal, where we observe events at different 
time locations, or spatial, where we observe events at different locations in space. 

