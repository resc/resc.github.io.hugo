+++
bigimg = ""
date = "2016-06-27T08:10:30+02:00"
subtitle = "Or how to stop worrying and just copy those utility classes."
title = "Osmosis"
tags  = [ "C#", "Go", "Golang" ]
+++

In my [previous post](/post/generate-all-the-things) I've shown how I build ad-hoc generators. You might also have noticed that there was a ```Disposable``` class. It does nothing fancy, it just wraps an [Action](https://msdn.microsoft.com/en-us/library/system.action(v=vs.110).aspx) in a [IDisposable](https://msdn.microsoft.com/en-us/library/system.idisposable(v=vs.110).aspx) interface so it can be used in a [using](https://msdn.microsoft.com/en-us//library/yh598w02.aspx) block. There's some benefits to that. The action will be executed even if there's an exception in the using block, and the action will only be called exactly once. Even in multi-threaded scenarios. 

So it's a generic, useful class, and it's tiny and trivial to type or copy.
Now, should it be in some utils project, that will be included in every solution? 

[Go proverbs](https://go-proverbs.github.io/) says no. Specifically it says:

_A little copying is better than a little dependency._ 

And I happen to agree. Dependency management is busy-work. It's incidental complexity that comes from a need to organize and compartmentalize your code to make it managable. Don't get me wrong, it's very much a necessary evil and its necessity grows with the size of your codebase. But it is evil nonetheless. See the [Node.js](https://nodejs.org/) [left-pad debacle](http://www.theregister.co.uk/2016/03/23/npm_left_pad_chaos/) for proof. (And go read [Haney's post](http://www.haneycodes.net/npm-left-pad-have-we-forgotten-how-to-program/) too, he said it better than I can).

