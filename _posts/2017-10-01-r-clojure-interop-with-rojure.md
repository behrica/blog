---
title: "R Clojure Interop With rojure"
layout: post
date: '2017-10-01 11:32:49'
published: yes
tags:
- clojure
---

# New Clojure library 'rojure' released #

I am proud to announce  the first version of "rojure",
a R-Clojure bridge using [clojure.core.matrx](https://github.com/mikera/core.matrix) and 
clojure.core.matrix.dataset for data exchange between R and clojure.

Is is a continuation of the 'rincanter' project.

* <https://github.com/jolby/rincanter>
* -> <https://github.com/skm-ice/rincanter>
* ->     rojure

but replacing the dependency to "incanter" and its data structures with dependencies on "core.matrix".

So the name change from rincanter to rojure was needed.

The code is now completely independent from incanter, but can be of course used from incanter (> 1.9.0)
as well.

The first version is released on clojars:
<https://clojars.org/rojure>

As it is only a small modification from rincanter, it should be rather stable already.
