+++
date = '2025-10-19T12:00:00-04:00'
draft = false
title = 'On LLM-based TUI coding assistants'
+++

# Introduction

## Backdrop

Over the past few months, it seems that Big AI&trade; has finally listened to us terminal users. As someone who exclusively uses nvim + tmux, I did (and continue to) not care about most of the AI-in-IDE assistants, since most of them seemed catered to VS Code (and its forks) or JetBrains users. Neovim has plugins that offer similar functionality such as Avante, and while they work great (so I've heard), they're all entirely community-driven (but it is worth mentioning that Microsoft did release a vim plugin for Copilot, which a community member kindly ported to neovim and continues to maintain to this day).

When I was at Google Cloud Next, there were many announcements on how Gemini "meets your developers where they are", so I was very confused (and mildly annoyed) that apparently the devs using the terminal didn't count. When I talked to the product manager of Gemini at Next, he mentioned there were no plans to implement a CLI coding tool. This was also roughly around the time that OpenAI released Codex, their competitor to Claude Code--which had been available for quite a while, and one of the few AI tools I actually use and to some extent, trust. So when Google announced the early access for the Gemini CLI a month or so later, I was confused and skeptical--how, in one month, did we go from no plans to an early-access preview?

## The plan

With Google's release, I decided to try out all of them so that I could pick one. The tests would not be the most scientifically rigorous, but relatively informal: I would work on my projects, and if I found that I was either stuck, or solved a problem in about 30-45 minutes, I would see how each of the CLI tools performed. Although my projects are by no means large (about 5k-10k lines of code each), they are realistic for most devs trying them for their side projects. I decided I would give the tools a variety of tests across the languages I use (Python and Rust) and varying difficulties. The main difference between these tests and ones that others have performed is that because I don't use AI unless I need to solve specific issues, the queries I give them typically require a fair amount of domain knowledge. This means that the tests will require the LLMs to use both the data they trained on, as well as the code I'm editing, to reason and suggest edits.

Before I discuss the tests, we should discuss my expectations. Except for the ones marked easy, the problems I gave these tools are generally speaking, pretty hard because of the need for domain knowledge (by domain, I mean the specific problem I'm working on, not in a broader sense). For those problems, I do not necessarily care about a fully working solution, just something closer to it from where I am, written in an idiomatic way. Aside from correctness, UX is a big part of my decision-making: how good is the TUI, how seamless is authentication, how well does it show progress updates, etc. For most problems, I tell the LLMs what file needs to be edited, and broadly, it seems to be very helpful for them as a starting point.

Of course, all of this is based not on data, just on personal anecdotes. You should take everything I say with a grain of salt and view it with the same level of skepticism as anything else, and form your own opinions. My overall belief remains that AI *can be* moderately useful *at times* to developers who have experience, know how to design systems, and critically examine the code it produces, and that it actively impedes newer/junior devs' progress and gives them a false sense of competence.

# The tests

## PDF parsing

I'm working on a RAG application for my Zotero library, and I'm writing the entire thing in Rust. One of the components of my system is a PDF parser that's somewhat tailored to `pdflatex`-generated papers (which is common in CS). Rust does not have high-level libraries to parse PDFs: the most convenient crate is `lopdf`, which does the heavy-lifting of converting the binary parts of the PDF into usable Rust types and providing iterators for the various objects in the PDF.

One of the peculiarities of my parser is that I wanted it to ignore tables and figures: the idea being that broadly speaking, they do not provide much additional information beyond what the text states (this varies by field, of course), and I do not believe they would be useful context for answering questions. I had already written a function in my parser to detect likely boundaries for tables[^1], which included a helper function to get the operands for operators in the PDF's content stream. My prompt here was relatively simple: use similar logic to ignore images. Images are not always represented in the content stream, and are usually marked by a `/Im` command (at least, `pdflatex` seems to prefer them). This immediately makes finding images easier, but there's also the task of looking around this position to heuristically determine the caption (and ignore it as well). The heuristic here is simple enough: we need to look at the vertical distance between the caption and other text blocks: if it's "large enough", we ignore the text until that paragraph.

Broadly speaking, the tools did poorly:

* Codex wrote a function that not only wouldn't work, but had unused variables, duplicated code, and did not handle escape sequences (which another function I had would've handled, but it wasn't even necessary: it just chose a more complicated design). When I pointed these issues out, it seemingly in an act of indignance, spit out what is possibly the worst Rust code I've ever seen. I didn't bother checking it for correctness because it was unreadable anyway.
* Claude wrote a function that ignored images correctly, and it even handled inline images (a case I didn't even think about, because I was too focused on images in the XObject dictionary). However, when I asked it to also ignore captions, it had a ton of trouble. My test PDF had two images, with text in between them. One of the captions was also multi-line (which in pdflatex means you have two `TJ` blocks). Claude spent maybe 15 minutes trying, failing, and debugging to get both of the captions to be ignored while not ignoring the rest of the text. Eventually, I put it out of its misery. However, Claude did write the most comprehensive tests of all three.
* Gemini ran into some compilation errors, and in its efforts to fix those issues, introduced new errors by changing completely unrelated code (as an example, it changed the keys of a hashmap I had that was responsible for handling ligatures. For some reason, it removed a backslash from the keys, which was needed to escape the backslash used in the key itself, and seemingly did not understand the compilation error).

A common theme was that these tools either don't have the ability, or refuse, to backtrack to an earlier version of the code. This means that if they start off on a problematic path (such as one with compilation errors), they tend to double down on that approach. This leads to code that gets less and less maintainable each iteration.


[^1]: In PDFs, text is represented via operators that specify the font, the font size, and the specific location where the text is. An additional complication is that `pdflatex` handles kerning, which means words will often get split up based on the font used. For tables, my heuristic was to look for graphics (to draw borders) and to look at operators nearby to determine if `pdflatex` is attempting to align columns.

## Improving an ML model

I was working on a model that used an autoencoder with a modified loss function. My problems were twofold: the results were very inconsistent across repeats (more than the usual variance caused by stochasticity), and it wasn't doing as well as it should have. On my own attempt at solving this, I spent an hour trying various changes, and eventually fixed the latter problem. While the three tools had varying levels of success, what I found interesting was the UX differences as I interacted with them.

* Codex spent a _lot_ of time thinking. For several minutes at a time, it would think, but wouldn't show me what its reasoning process is. This means that even if I wanted to, I couldn't help it if I saw it was stuck on an issue I've previously faced and resolved. This also results in a lot of spam on the terminal with "thinking for x seconds". Codex also very consistently across all my tests, 429d itself (a problem that newer versions seem to have fixed). All of that said: the resulting solution it came up with was the best among the three, _by far_. While it didn't do as well as my solution did, it was more consistent.
* Claude Code spends a lot of time in experimenting, and that was most apparent for this problem. It did what felt like an ablation study, changing one parameter at a time and then running the test suite. While it made some attempts and got close at times, it kept getting poor results. Eventually, it concluded that the dataset was the issue because it was too imbalanced.
* Gemini (at least the earlier versions) seemed to embed a terminal in its TUI, which led to flickering when there was a lot of text changing or scrolling. It's pretty distracting visually. However, it has a really nice text input box where the up arrow key seems to work contextually: in a multi-line input, it moves across lines, but if it's at the top, it goes to the previous input. Gemini also 429d itself on this problem, and did not get a solution at all.

## Refactoring

I haven't used these tools for refactoring very often (I just write good code from the start, *duh*). I typically only refactor if I find that crates I write are not ergonomic, or I can't find things like structs or variables (which usually is a sign that it's not obvious enough), or the code just feels like it could read better and be more polished.

There was a point at which I had several constants, strings, etc. that were all over the place, my structs weren't super consistent, and some code was in non-obvious places (I would spent *seconds* looking for it). I had the three tools refactor my crate. They were given broad advice to make things more idiomatic and maintainable. In general, the three tools did okay here; they made some objectively good changes, and some other changes that I wouldn't have made (some good, some not). The common theme was that I had to decide what refactors I did and didn't want. 

In general, I use the CLI tools not on autopilot (so-called "vibe coding"), but I have them ask for permission for *everything*, and I ask questions if I'm not sure why something was done a certain way[^2]. The nice thing about this, once you get past the annoyance of having it ask you for one-line changes, is that you can steer them in ways you like. For example, Claude in particular would often place imports locally (i.e., near the code, in the function or module it was being used). I tend to find this distracting in general, and much prefer all my imports at the top of the file--but after I told it the first time, it always placed imports at the top (at least for that session, anyway).

[^2]: I absolutely do not trust the quality of the code or "decisions" made by LLMs, at least not as of writing. This is why I carefully check every change they make and every command they run. There have been very limited times when I did use them in autopilot, mostly if I had already spent an hour or so debugging something and was fed up. See below.

## KerasHub

I volunteered to add Microsoft's Phi-4 to KerasHub (formerly `keras-nlp`). I had done most of it, and was just running tests at this point. One specific test in particular kept failing, despite my repeated attempts. And this was definitely a bug in my code--it's just that I had so few lines of code that it was not obvious why. At some point, I just committed my changes and let the tools try one at a time, this time in autopilot (except for running commands). I figured that the worst case would be me needing to `git restore .`.

* Codex was by far the least efficient, because it spent *so long* thinking, only to end up at a non-working answer. I killed it pretty early because the lack of intermediate reasoning steps was quite annoying. If "thinking" took minutes, was it because a network connection failed and it doing an exponential backoff? Was there some weird bug? Who's to say--certainly not Codex.
* Claude Code had, in my opinion, the best strategy. It began by running the test, looking at the code and the modules I had subclassed, and then used a mixture of print-debugging and inspecting object state to arrive at conclusions. Importantly, it did these experiments in a separate file--this made it quite easy for it to make changes, and for me to take a closer look at what it was trying . It ultimately did not fix the bug, but its experiments did give me new ideas for what could be causing it, and directly led to me fixing that test. From that perspective, I call this a win.
* Gemini had the opposite strategy: it ran the test, looked at the output and my code *once*, and declared that not only was my code perfect, *the shared utility functions* were buggy. You know, the shared utilities that the dozens of other models in KerasHub's repo used without issue. This was too wild of an initial guess for me, so I did not take it further.

# PR review tools

Some time between when I started writing this and when I finished, most model providers also added GitHub bots that you could add to your repo to provide PR feedback. Naturally, I tried all of them with my RAG project.

* Copilot was, *by far*, the worst. Not only did it consistently not seemingly know Rust very well (for example, it did not know that `std::env::set_var` was now `unsafe`), hallucinate library functions from their older versions, its code review comments and suggestions were also terrible. The *one* time I accepted one of the changes it suggested, it broke a completely different part of the system and I had to clean up its mess. This, and the lackluster Copilot performance in neovim, means I will never purchase their subscription.
* Claude Code's reviews are generally okay. They're sometimes too polite or downplay particularly bad issues that it catches, which is not a good thing. However, it could at times be a stickler for my project's style guide--and I really appreciated that.
* Gemini was generally unbothered with the style guide, unless there was something particularly bad. However, Gemini has caught several major issues in my code. It did have the Copilot issue where it hallucinated functions from earlier library versions, but that has gone down over time, and in general, its recommendations are pretty solid. More recently, Gemini seems to be particularly nitpicky in its PR comments when describing bad inefficiencies, poor code style, or logical errors. Personally, this is my favorite part of it, and it's why it's the only LLM-based PR review tool I'll ever recommend. In code review, I do not want a sycophant; I want it to point out issues, and if the issue is bad, I *want* it to tell me off. Here's one comment that really stuck with me:

> This section is highly inefficient. You fetch records with zero embeddings into `zero_batches`, then extract their keys, only to fetch *all* items from the database again with `get_lancedb_items` and filter them by those keys. This will perform very poorly on large databases.
>
> The `zero_batches` should already contain all the necessary data, including `pdf_text`. You can convert `zero_batches` directly to Vec<ZoteroItem> and proceed. The comment on line 161 might be based on an outdated assumption.

It was correct, suggested a *significantly better* code change, and also called out the assumption I had made (it wasn't outdated, it was plain *wrong*). In my defense, it was 3am when I wrote that code, and I really wanted to finish and go to bed. This and a few other comments it has left have cemented its place in my development workflow.

# Closing thoughts

I have opinions about AI-assisted coding. While Copilot has only worsened AI's reputation in my mind (but weirdly, Codex is overall okay, so I don't know why Copilot feels so kneecapped), my overall impression ranges from neutral (due to Claude Code) to, very recently, cautiously optimistic (Gemini).

I do not think AI improves developer productivity as much as the companies offering such tools (or unwitting CEOs) would like you to believe, and I am quite firmly of the belief that if AI does make someone 10x better, they were a 0.1x dev to begin with. I also do not think most devs using AI give it nearly enough (if *any*) scrutiny. All that said, I think for senior developers (by "senior" I mean people who have been programming, professionally or otherwise, for at least 5 years [^3]), AI can sometimes be useful *as a tool, but not a copilot*. In my experience, I would guess that using something like Claude Code/Gemini have improved my productivity by about 10%. This is not an indictment on LLM-based tools: a 10% improvement is pretty big. If my editor loaded 10% faster, I'd be *ecstatic*. My issue isn't with AI itself, but with the irresponsible and borderline deceptive marketing surrounding it.

[^3]: I do not care to debate this definition. If you prefer the term "intermediate", read it as such.


## Cost

It is important to talk about cost when discussing tools like this, because the companies advertising them would prefer you just hand over your credit card and let the LLM work on its own while using up tokens. My personal experience has been that all three of these tools are actually surprisingly cost-effective (with the only exceptions being the Claude Opus series and OpenAI's o3-pro).

* Whenever I've used Claude Code to help me debug an issue, which involves a lot of running tests, updating files, and retrying, I've found that it almost always costs around \$0.60 to \$1 or so. This has meant that even though I initially loaded my Anthropic account with \$25 of credits months ago, I still have half of it left. In general, I find Claude Code to be very much worth it--but note that I use AI tools very sparingly.
* The very first time I used Codex, it used up 750,000 tokens, spent 5 minutes thinking, and then 429d itself--but it had a working solution, so I'm not sure why it kept going. The entire thing cost me only $0.60 (it was using o4-mini), which is reasonable given how many tokens it used.
* Gemini in general is quite attractive for its cheaper pricing. Another bonus for me is that you can have the bill for Gemini CLI be added to your Google Cloud bill, so it's one less account/bill to think about.
