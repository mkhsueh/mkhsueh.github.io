---
layout: post
title: "XML Namespacing"
date: 2015-01-27 00:43:20 -0800
comments: true
categories: 
published: false
---

Namespacing design in general can create unexpected ripple effects in a developer's workflow. The impact might range from requiring considerations in initial application design to deleterious effects on productivity and code bloat. In the most unfortunate situation I recall, one of my customers had ended up with a very fragmented XML schema. Different tech units siloed themselves and ended up re-creating many of the same business entities in their own team's namespace. By the time they started to notice the consequences, a refactoring of all their systems had likely become too expensive. Working with their existing codebase, I once spent an entire day writing thousands of lines of namespace conversion code just to implement a simple business feature. 

