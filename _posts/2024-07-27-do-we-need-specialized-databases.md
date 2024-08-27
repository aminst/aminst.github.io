---
layout: post
title: Do we need specialized databases?
cover-img: assets/img/til.jpg
tags: [databases]
category: TIL
---

Not completely unrelated “One Size Fits All” An Idea Whose Time Has Come and Gone (Stonebraker )

The question is why do we need to build a new database for different domains. For example, TigerBeetleDB is a database for financial accounting. 
Do we need a database to store vectors for LLMs? Vector Databases

I think it depends; if you need a completely new structure to handle the type of problems that you are solving, go for it.
If some existing solution can handle what you are looking for, then what is the point in building something new from scratch?
The main problem is to identify which of the two cases holds for you. That might be not trivial.