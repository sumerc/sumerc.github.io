---
title: "Sidenotes demo"
description: "Footnotes that sit in the margin, next to the sentence they belong to."
date: 2026-11-01T09:00:00+03:00
draft: true
tags:
  - meta
---

This is a draft post to try out margin notes. Write a footnote the normal Markdown way, and on a wide screen it shows up beside the paragraph instead of at the bottom.[^1] On a phone, nothing changes: the notes stay at the end of the post like before.

Hover a number, or the note itself, and both light up so you can see which one belongs to which.[^2] Click a number and the note stays highlighted for a moment, which is handy on a trackpad.

## Why bother

Footnotes at the bottom of a post are a round trip: scroll down, read, scroll back, find your place. A margin note is zero cost to glance at and zero cost to ignore.[^3] That makes it a good home for the stuff that would otherwise go in a parenthesis: a caveat, a source, a "well, actually".

Notes can hold more than one line, and a bit of markup. Code works: the `runtime.SetCPUProfileRate` call is a classic example of something you want to warn about without derailing the paragraph.[^4]

```go
func main() {
    pprof.StartCPUProfile(os.Stdout)
    defer pprof.StopCPUProfile()
    work()
}
```

When several notes land close together they stack instead of overlapping, so a dense paragraph still reads fine.[^5] Even a second note right after it.[^6] And a third, to make the stacking obvious.[^7]

You can also reference the same note twice.[^2] Both numbers point at the single margin note, and hovering either one highlights it.

## Below the fold

A long tail of plain text, so there is something to scroll past and so the last note has room.[^8] Lorem ipsum is boring, so here is a reminder instead: the profiler signal handler runs on whichever thread happened to receive `SIGPROF`, which is why the stack you see is not always the stack you expected.

That is all. If this looks right, the draft flag comes off and the CSS stays.

[^1]: This is the first note. It is just a sentence, which is what most notes should be.
[^2]: This note is referenced twice in the post. Hovering either number highlights this same note.
[^3]: Sean Goedecke's blog does this and it is where I stole the idea from: <https://www.seangoedecke.com/>.
[^4]: Setting the rate above 100 Hz on Linux used to be silently ignored before Go 1.18. See the `inside-the-go-cpu-profiler` post for the long version.
[^5]: Note five. Short.
[^6]: Note six, which is a little longer so you can see that the one below it gets pushed down rather than drawn on top.
[^7]: Note seven. Still readable.
[^8]: Last note. Nothing after this but the footer.
