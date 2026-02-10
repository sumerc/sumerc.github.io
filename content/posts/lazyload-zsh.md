---
title: "Improve zsh startup time via lazyload"
date: 2021-12-31T11:29:57+03:00
draft: false
tags:
  - shell
  - performance
---

# My shell is slooow!

This is my first blog post for years. I have been finding excuses for not writing one over 
the years, but I feel this is the right time.

I was coding as usual when I started something odd with my shell. I have done few adjustments
to my `.zshrc` file and out of a sudden it became very slow to spawn a new shell. As a person
who has a obsession with performance, it became my duty to find out the problem.

# The culprit

I have done some measurements:

{{< highlight shell >}}
> time /bin/zsh -i -c exit
/bin/zsh -i -c exit  0.30s user 0.41s system 34% cpu 2.077 total
{{< / highlight >}}

2 secs just for spawning a new shell? There is <i>really</i> something odd. Well at this point,
I thought using some kind of profiler on my Mac like samples, but then I decided it will be simpler 
to go over the things on `.zshrc` file comment out some of them to find out what is causing this annoying 
lag.

After a while, I found 3 things:

{{< highlight shell >}}
[ -s "$NVM_ROOT/nvm.sh" ] && \. "$NVM_ROOT/nvm.sh"
[ -s "$NVM_ROOT/bash_completion" ] && 
    \. "$NVM_ROOT/bash_completion"

export PATH=$RVM_ROOT/bin:$PATH
source $RVM_ROOT/scripts/rvm

export PATH=$PYENV_ROOT/bin:$PATH
eval "$(pyenv init --path)"
{{< / highlight >}}

As you can see, I am frequently using `nvm`, `rvm` and `pyenv`. That is because my day-to-day job requires jumping between 
different versions of Node/Python/Ruby currently.

When I comment out above code, the startup time decreases to `0.153` from `2.077` :)

# lazyload extension to the rescue

Most of the time, I don't need these stuff to be initialized at all. However, I need them
when I need them.

There is a extension called `lazyload` that takes a command and do the initialization only once
when you need them. Within few minutes, I arrived at following code:

{{< highlight shell >}}
lazyload nvm -- '[ -s "$NVM_ROOT/nvm.sh" ] && 
    \. "$NVM_ROOT/nvm.sh" 
        [ -s "$NVM_ROOT/bash_completion" ] && 
    \. "$NVM_ROOT/bash_completion"'

lazyload rvm -- 'export PATH=$RVM_ROOT/bin:$PATH
    source $RVM_ROOT/scripts/rvm'

lazyload pyenv -- 'export PATH=$PYENV_ROOT/bin:$PATH
    eval "$(pyenv init --path)"'
{{< / highlight >}}

Which was exactly what I needed!

My full `.zshrc` file: https://github.com/sumerc/dotfiles/blob/main/zshrc

And the `zsh-lazyload` extension: https://github.com/qoomon/zsh-lazyload

# One final thought

Right now, I am really wondering why these tools do not provide this kind of lazy-loading by default.
It seems far better practice especially for people that have to install lots of different tools.
