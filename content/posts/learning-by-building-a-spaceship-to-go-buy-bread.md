---
date: 2026-06-07
layout: post
title: "Learning by Building a Spaceship to Go Buy Bread"
subtitle: GSimple
---

Looking through private repos, I came across **gsimple**, a project I built late last year to brush up on old knowledge, capitalize on new learnings, and play with Gemini & Cursor. As the name suggests, it is a super simple API for managing products made in Go. The goal here wasn't the app itself, but to review and learn several new things about how an application is deployed today.

## It's not just the code, it's the "surroundings"

Go appeared on my radar thanks to my time at Mercado Libre. I immediately thought it was great as a language because of its simplicity, speed, and concurrency support. It also reminds me of C++; I wrote my System Engineering thesis on C++. Since then, I've built many toy projects; I call them "cardboard projects." Generally, they helped cement some knowledge, but many times they didn't go beyond a `main.go`. With **gsimple**, I intentionally set out to go further, with a lot of help from Gemini & Cursor in this case, which are the tools I also chose for testing. I didn't just want the code to work; I wanted to understand how it feels to move that piece of code through a real pipeline.

## What cost the most this time

*   **Storage abstraction:** Using interfaces to be able to switch from an in-memory map to a MySQL database without breaking all the logic.
*   **The YAMLs:** I'm barely touching the surface of Kubernetes using Helm, Kustomize, and Skaffold. Moments of deep frustration without understanding errors, a lot of back and forth with an LLM here to understand what was happening and how it works. Then, when everything comes up, what a great feeling.
*   **Infrastructure as code:** Same as Kubernetes, an expedition into the world of Terraform; overkill for this project size, but also very interesting and with a lot of room to keep learning.

## More learning

Many improvements pending:

1.  **Error handling can be improved:** Learning to wrap errors so they are useful in production is something to keep iterating on my end.
2.  **Kubernetes is a monster:** This project is the closest thing to building a spaceship to go buy bread.
3.  **The tests:** There are tests, but coverage is lacking, especially in edge cases. Doing real integration tests with the database remains a challenge that requires more time in my case.

### What is missing?

It's far from being "finished":

*   **Security:** Right now it's an open door. Some type of Basic Auth or JWT should be added.
*   **Observability:** The placeholders for OpenTelemetry are in the Helm chart, but the real implementation in the code is a pending step to see traces and metrics.
*   **Documentation:** OpenAPI specification included, originally with the idea of trying Design-First.

### Important Note on Git History and Security

The Git history of this repository was intentionally purged before making it public. At some point, the early history mistakenly contained some keys of a defunct infrastructure, but it felt right to remove them. Also, the commit message history wasn't something to be proud of; this was before I discovered Conventional Commits and understanding that commit messages matter, even in toy projects.

### Conclusion

Another simple record of learning. An LLM can write code very fast—Gemini and Cursor in this case. Something that doesn't leave as much record is the rest: infinite questions to ChatGPT in this case to understand how the parts work. Definitely an example where the most interesting part is the journey rather than the final product.

[Link to the source code repository](https://github.com/mamcer/gsimple)