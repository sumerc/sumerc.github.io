---
title: "lazyload commands to improve `zsh` startup time"
date: 2021-12-31T11:29:57+03:00
draft: false
categories: ["shell"]
---

# My shell is slooow!

This is my first blog post for years. I have been finding excuses for not writing one over 
the years, but I feel this is the right time.

I was coding as usual when I started something odd with my shell. I have done few adjustments
to my `.zshrc` file and out of a sudden it became very slow to spawn a new shell. As a person
who has a obsession with performance, it was my duty to find out the problem.

# The culprit

I have done some measurements:

{{< highlight shell >}}
> time /bin/zsh -i -c exit
/bin/zsh -i -c exit  0.30s user 0.41s system 34% cpu 2.077 total
{{< / highlight >}}

`2` secs? There is <i>really</i> something odd. Well at this point I thought using some kind of
profiler on my Mac like samples, but then I decided it will be simpler to go over the
things on `.zshrc` file comment out some of them to find out what is causing this annoying 
lag.

After a while, I found 3 things:

{{< highlight shell >}}
[ -s "$NVM_ROOT/nvm.sh" ] && \. "$NVM_ROOT/nvm.sh"  # This loads nvm
[ -s "$NVM_ROOT/bash_completion" ] && \. "$NVM_ROOT/bash_completion"  # This loads nvm bash_completion

export PATH=$RVM_ROOT/bin:$PATH
source $RVM_ROOT/scripts/rvm

export PATH=$PYENV_ROOT/bin:$PATH
eval "$(pyenv init --path)"
{{< / highlight >}}

As you can see, I am frequently jumping between different versions of Node/Python/Ruby. That is
because my day-to-day job is basically writing/learning profilers/tracing tools for different 
languages. 

When I comment out above code, the startup time decreases to `0.153` from `2.077`.

# `lazyload` extension to the rescue

Most of the time, I don't need these stuff to be initialized at all. However, when I need them
I need them. While I can write a function to init this stuff manually, I found that solution 
dirty. And I also thought I could not be the first one facing this.

There is a extension called `lazyload` that takes a command and do the initialization only once
when you need them. Within few minutes, I arrived at following code:

{{< highlight shell >}}
lazyload nvm -- '[ -s "$NVM_ROOT/nvm.sh" ] && \. "$NVM_ROOT/nvm.sh"  # This loads nvm
    [ -s "$NVM_ROOT/bash_completion" ] && \. "$NVM_ROOT/bash_completion"  # This loads nvm bash_completion'

lazyload rvm -- 'export PATH=$RVM_ROOT/bin:$PATH
    source $RVM_ROOT/scripts/rvm'

lazyload pyenv -- 'export PATH=$PYENV_ROOT/bin:$PATH
    eval "$(pyenv init --path)"'
{{< / highlight >}}

Which was exactly what I needed!

My full `.zshrc` file: https://github.com/sumerc/dotfiles/blob/main/zshrc

And the `zsh-lazyload` extension: https://github.com/qoomon/zsh-lazyload

# One final thought

Right now, I am really wondering why these tools provide this kind of lazy-loading by default.
It seems far better practice. Especially for people having to install lots of different tools.
