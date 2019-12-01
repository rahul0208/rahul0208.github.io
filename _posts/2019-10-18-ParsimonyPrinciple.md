---
layout: post
title: Parsimony Principle
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [design]
---

>  *The most simple of two competing theories should be the preferred one, and that entities should not be multiplied needlessly*

I have been building systems for over a decade now. When we build systems there are principles like **Keep It Stupd Simple** or the **You Aint Gonna Need It**. It is important to enforce these principles. These principles are not only applicable to Software development. Simplicity is a general philosophy which is applicable to Science, Technology, Maths etc. Various philosphers have enforced the principle again and again :

> *Everything should be made as simple as possible, but not simpler* – Albert Einstein
>
> *Entities must not be multiplied beyond necessity*- Occam Razor

But yet often while in the middle of building things the concept of simplicity is overlooked upon. 

 The overlook is quite evident in the many hype cycles of software development. I started my career at a time when J2EE was pitched as the de-facto standard of application development. But then there was dissent over the vide array of technologies packed together. This lead to the rise of frameworks like Spring, CDI, Guice, micro-profile.

 Then again after a while people pitched Big data to be the game changer. But I then the Hadoop ecosystem developed by Google was quite complicated for the other organizations to support. This lead to Spark and other related technologies, none of them being a cake-walk. 

 Similarly NoSQL came to the ecosystem. They started saying No-to-SQL, but soon the adopters the fallacy. It wasn't easy to support NoSQL and the ecosystem became not-only-SQL. Eventually RDBMS stores developed as JSON store, and now people are back to RDBMS ecosystem. Enterprises are still asking for RDBMS grade features like transitions in NoSQL stores like Mongo.

 These days we are talking about Kubernetes. But working with K8s is not easy. It is a one big mammoth. It has its own learning curve. There are so many things involved. I see many customers jumping to K8s wagon for their ecosystems. But it is important to understand the K8s is developed by Google, just like previous technologies like NoSQL, Big data etc. Google would have use case to support K8s architecture. But there are limited enterprise of that scale. Depending on the ecosystem where it is being absorbed its mileage will vary. It may be quite simple for Google to work with K8s. But this can't be said about others. 

 Application development practise like XP, Agile repeatedly enforce keeping things simple. Yet again and again we as architects over-engineer our enterprise architecture.



