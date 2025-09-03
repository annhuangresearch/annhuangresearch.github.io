---
title: "One hot encoding"
layout: post
date: 2024-08-13
tag:
- coding
category: blog
author: annhuang
description: 
---

One-hot encoding is a method to represent categorical variables numerically. Each category gets its own binary variable: 1 if the observation belongs to that category, 0 otherwise.

Computers only understand numbers, so categorical information needs to be converted.

- Avoids implicit ordering that happens if you use integers (0, 1, 2…) for categories.
- Prevents multicollinearity when done properly, because each variable is orthogonal to the others (if you leave out the reference category).