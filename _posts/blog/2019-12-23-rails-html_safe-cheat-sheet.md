---
layout: post
title:  "Rails html_safe Cheat Sheet"
date:   2019-12-23
category: blog
---

Rails uses [`ActiveSupport::SafeBuffer`](https://edgeapi.rubyonrails.org/classes/ActiveSupport/SafeBuffer.html) to prevent [cross-site scripting attacks](https://en.wikipedia.org/wiki/Cross-site_scripting). But when a string will or won't be escaped can be confusing. Part of this confusion stems from when the unsafe string is *actually* escaped.

After doing some heavy `#html_safe` work we thought it would be helpful to create a cheat sheet containing the most common use cases.

Cheat sheet entries reference the following variables:

```ruby
safe   = "<a href='#foo'>click here</a>".html_safe
unsafe = "<script>someMeanJavaScript()</script>"
```

| Operation                           | Result is `html_safe?` | Notes                                                                          |
|-------------------------------------|------------------------|--------------------------------------------------------------------------------|
| `safe << unsafe`                    | `true`                 | `unsafe` is escaped immediately                                                |
| `safe += unsafe`                    | `true`                 | `unsafe` is escaped immediately                                                |
| `unsafe << safe`                    | `false`                | escaped when view is rendered                                                  |
| `unsafe += safe`                    | `false`                | escaped when view is rendered                                                  |
| `"Amaaaaaazing content: #{safe}"`   | `false`                | escaped when view is rendered                                                  |
| `safe.to_s`                         | `true`                 | &mdash;                                                                        |
| `safe.to_str`                       | `false`                | More on [`#to_str`](https://blog.bigbinary.com/2012/06/26/to_str-in-ruby.html) |
| `safe.safe_concat(unsafe)`          | &mdash;                | raises a `SafeConcatError`                                                     |
| `safe.safe_concat("foo".html_safe)` | `true`                 | &mdash;                                                                        |
