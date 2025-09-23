---
layout: post
title: Arcane Code and AI
subtitle: Playing with Gemini CLI, ChatGPT and Claude Sonnet
---

A while ago, I was sorting through old (very old) files. I found the assignments I did in college and literally my first programs or codes written, first in Pascal and then in C++.

Review them with VS Code, nostalgia, re-archive, forget them again.

A few days ago I remembered those codes and now the idea of ​​playing with AI, why not?

Among the programs, I found the final project for Algorithms I (1999), which consisted of generating a simple C++ program that encodes and decodes a message in limited Morse code. Using the statement as a starting point, I asked Gemini (CLI), ChatGPT (Web), and Claude Sonnet 3.5 (from VS Code Copilot) to create a Golang implementation. Just to clarify that all three are free versions, currently I'm not paying for any AI services.

Three solutions that differ in lines of code and approach. All three are valid, although, for example, Gemini doesn't take whitespace into account, so a message consisting of different words separated by spaces doesn't yield the original message when encoded and then decoded. The limitation here lies in the input Morse code, which only takes letters into account and a limited set of symbols. But a difference here that both chatgpt and claude found a workaround on their own to avoid this situation.
<!-- 
[Claude](https://github.com/mamcer/arcane-code/blob/main/algoritmos-1/claude/main.go)  
[ChatGPT](https://github.com/mamcer/arcane-code/blob/main/algoritmos-1/gpt/main.go)  
[Gemini](https://github.com/mamcer/arcane-code/blob/main/algoritmos-1/gemini/main.go)   -->

Seconds to obtain the solution compared to likely hours of analysis and design of the solution, coding, and then surely some debugging to fix errors. I really like the Claude version, closely followed by ChatGPT for the way it handles edge cases.

Another test I ran was using a program called `ROMANO.PAS` originally written by me in 1998, which, given a Roman numeral, returns its numerical representation.

<!-- [roman.go](https://github.com/mamcer/arcane-code/blob/main/programacion-1/roman.go) -->

All of these examples are uploaded to a repository where you can also see my first steps creating and destroying objects. Some are practical work, some are personal learning and practice, seeking to understand more and more of this for my new and fascinating world of programming.

Originally written in Turbo Pascal and Borland C++ on Windows NT 4 in college and then Windows 98 on my first PC... `I was there 3000 years ago...`

<!-- [windows-nt-4.0](https://betawiki.net/images/4/4c/Windows-NT-4.0.1381.1-Desktop.png)
> betawiki.net -->

With AI, I believe the barrier is lower than ever for programming and generating quality solutions in record time. One question I ask myself is whether someone starting their career in IT today is really learning to program and then design systems. These same tools can generate a time and result expectation that is incompatible with truly instilling that knowledge in a person. In other words, the temptation is very strong to write a prompt, have an AI generate all the code, see that the expected output is obtained, and move on to the next problem or issue to be solved without pausing to understand what it did. Maybe the real question here is: is it necessary today to stop and seek to acquire that knowledge in the first place? If the answer is that we will always have an AI available, then perhaps we no longer need to have great knowledge or depth of solutions; we just need to focus on writing quality prompts until an AI solves any problem that may arise. Here we have to see where the quality of the prompt begins to be affected by not knowing the details of what we are trying to build or solve.

For example, historically, my learning approach was to learn by doing, little system designs on paper, POCs, very simple programs that allowed me to test edge cases, for example, or to cement in my mind something I read in a book or post, or imagined was possible but had no confirmation. Different ways of working that today seem deprecated, almost obsolete.

I also believe that, as they say: "everyone dies in their own way," in their own style. In my case, I found a super senior in AI available through multiple channels. Whether in the IDE, CLI, and Web to complete monotonous project tasks, which are necessary but perhaps repetitive or don't provide much new learning value, or other very valuable ones, showing new and often better ways to implement a solution. Last but not least, something I've always liked is having as few black boxes as possible. At least in general terms, understanding what they do, how the different technologies I use or interact with work. AI is absolutely brilliant for that. Spontaneous questions to understand or confirm assumptions about how a particular technology or component works, I find it fascinating and very addictive to pull that thread and continue learning related topics.

What a great moment for anyone who wants to learn and understand how things work.