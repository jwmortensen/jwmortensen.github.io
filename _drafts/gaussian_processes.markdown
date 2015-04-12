---
layout:  post
title: "Gaussian Processes" 
date: 2015-04-03 20:49:00
categories:
- gaussian processes
- frequentist statistics
tags: featured
---

Spatial data presents a challenge to traditional data analysis methods because it is highly
correlated and is typically nonlinear. While other methods, such as natural splines or polynomial
regression, can be used to model nonlinear relationships, additional measures would be required to 
compensate for collinearity. One alternative method that handles both the issue of nonlinearity and
collinearity beautifully is a Gaussian process.

### What is a Gaussian Process?

A Gaussian process is a form of stochastic, or random, process. We can think of 
stochastic processes as a potentially infinite collection of random variables over a domain
\\( \mathcal{T} \\). This domain could be temporal, where we observe events at different 
time locations, or spatial, where we observe events at different locations in space. 
With a Gaussian process we are assuming that each point within the domain follows a normal
distribution, and therefore the joint distribution of all of the points is simply a multivariate
normal distribution.
