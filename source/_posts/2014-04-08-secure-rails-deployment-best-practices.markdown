---
layout: post
title: "Secure Rails deployment: Best practices"
date: 2014-04-08 00:23:23 +0200
comments: true
categories:
---

``` ruby Discover if a number is prime linenos:false
class Fixnum
  def prime?
    ('1' * self) !~ /^1?$|^(11+?)\1+$/
  end
end
```