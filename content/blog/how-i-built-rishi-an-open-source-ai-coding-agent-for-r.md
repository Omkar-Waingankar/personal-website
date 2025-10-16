+++
title = "How I built Rishi, an open-source AI coding agent for R"
date = "2025-10-06"
author = "Omkar Waingankar"
description = "This blog post explores the product and engineering choices that went into building Rishi, an open-source AI coding agent for R."
+++

Today, I launched Rishi, an open-source AI coding agent for R. Building Rishi thus far has truly been a labor of love, and it felt necessary to peel back the curtain a bit and share why I started this project and what went into bringing it to life. If you're looking to learn more about Rishi as a product and try it out yourself, please visit [tryrishi.com](https://tryrishi.com).

<video style="max-width: 100%;" controls src="https://pub-21dd5a46fa1f46798859612fa3cf3a8a.r2.dev/rishi_demo.mp4"></video>

# A brief history of R and RStudio

If you were a serious data scientist in the late 90s or early 2000s, chances are you spent your day-to-day writing code in SAS. It was the de facto analytics programming langauge at the time, and was widely adopted in the enterprise, pharma, and government agencies.

However, by the late 2000s, a new player had emerged: R. This was an open-source, community-driven alternative to SAS with surging popularity from academia. New R packages were sprouting every day and quickly overtook SAS in feature richness.

Riding the wave of this challenger's success, RStudio was born soon after. It addressed the critical gap in the ecosystem which was a polished IDE that made package management, plot visualization, and code writing far easier. And in the original ethos of the R community, RStudio was open-source as well.

Today, R is an incredibly mature programming language and RStudio remains the IDE of choice for data scientists across academia and industry.

# The AI adoption curve

A massive paradigm shift in how code is written, reviewed, and deployed is currently taking place. Tools like Cursor and Claude Code can now index and reason about your project in real time, generate entire files from scratch, make precise edits to squash bugs, and even access web search to fill in knowledge gaps it lacks. These tools have fundamentally changed how developers work in 2025, making it possible to ship faster and tackle problems in hours, not weeks.

Despite this, AI-driven programming has not yet achieved ubiquity outside Silicon Valley or entrepreneurial circles. In particular, the plethora of work done by data scientists and researchers in academia has yet to be dramatically augmented with these tools. 

I know this firsthand because the idea for Rishi started with watching my girlfriend, a biology PhD student, give up on attempting to debug an obscure error herself and turn to ChatGPT for help (no shame at all, as I soon learned R errors are uniquely cryptic and unhelpful). She'd copy the error into the browser, wait for a response, copy that back into RStudio, find it didn't work, and repeat the cycle. 

It felt like I had turned back the clock about a year ago, and I immediately thought to myself "there has to be a better way".

# Alternatives exist, why not use them?

That initial observation led me to start having conversations with other grad students and data scientists in my network. I wanted to better understand where R practitioners from different backgrounds sat on the AI adoption curve.

Like my girlfriend, there were many folks from scientific backgrounds that sat earlier on this adoption curve. Their primary expertise was not software development, and they purely wrote code for the purposes of data analysis. They had a particular sensitivity to data privacy concerns and were reluctant to upload their datasets on the web. They would only occasionally turn to ChatGPT to debate statistical method choices, investigate new packages, or debug errors. And when they did so, they found results to be mediocre at best. 

Digging a little deeper, I learned that most frontier models aren't trained as heavily on R documentation and data analysis related tasks, leading to semi-frequent hallucinations of nonexistent functions and packages. Consequently, these individuals preferred to sit in the driver's seat and write most of their code in RStudio themselves, and seemed more resistant to the promises of agentic coding. 

Other though, were more adventurous and further along the AI adoption curve. Some tried Github Copilot autocomplete when it was first made available natively in RStudio, and were hooked. And then there were those who really stretched outside their comfort zone to adopt other IDEs like Cursor to test the cutting edge or try running Claude Code from within the Terminal tab in RStudio.

Over time, their usage matured and they arrived at similar conclusions to the first group. They were annoyed by the frequent hallucinations, tool calls for web search took forever, and autocomplete was really only useful for boilerplate helper methods. Eventually, they all came back to RStudio and adopted a more conservative approach to adopting AI in their day-to-day.

# How Rishi addresses all these concerns

The insights gleaned across these conversations were instrumental in making key product decisions that shaped how Rishi was eventually engineered.

First, I intentionally chose to build Rishi as an RStudio addin, not a standalone RStudio fork. This is because RStudio is by and far the most beloved IDE for R, people have muscle memory built around its interface, and there's an innate sense of trust in using this tool that has been maintained by the community for decades.

Second, Rishi would need to be open-source and architected to run entirely locally. Since concerns about data privacy were a recurring theme amongst all those I spoke to, I wanted to take special care to give users peace of mind that their confidential information was respected. Rishi's code is fully auditable, and it's designed so that messages, code, and data never touch a third-party server (other than the model provider APIs themselves, which don't train on API traffic).

Third, I needed to add explicit guardrails to prevent Rishi from hallucinating like other tools. I added support for a custom R help tool that grounds AI answers in actual package documentation. When Rishi needs to reference a function, it can pull up the official help files to ensure accuracy.

Fourth, and finally, Rishi's UX needed to be as simple and intuitive as possible. Even though the audience is other developers, I wanted to ensure that anyone could get started without having to read the usage guide or go through a bunch of setup steps. You just install the addin, enter your API key, and start chatting with Rishi.

# How Rishi works under the hood

Counterintuitively, most of Rishi is not actually written in R. There's only a couple hundred lines of R code in the entire project, and it exists purely to launch the addin and provide tooling that allows Rishi to interact directly with RStudio's interface via the `rstudioapi` package.

The frontend is written in TypeScript and React, which webpack bundles into a single static JavaScript file that gets shipped with the addin. The backend agentic loop is written in Go and compiled into a binary that's also installed alongside the addin. When you install Rishi from GitHub using:

```r
remotes::install_github("Omkar-Waingankar/rishi", subdir = "addin")
```

you're pulling down a pre-bundled package that contains the compiled frontend asset, the Go binaries for various OS targets, and the R launcher.

At runtime, when you launch Rishi with:

```r
rishi:::rishiAddin()
```

two things happen simultaneously: it opens the RStudio Viewer pane and renders the Rishi frontend, while also executing the compiled Go binary corresponding to your OS to spin up the backend server. The frontend then communicates with this local server to handle all the AI logic, tool execution, and conversation management.

I explicitly chose this architecture for a few reasons. The primary reason is because I'm admittedly more familiar with TypeScript, React, and Go than the R/Shiny ecosystem, so I could move faster and build something more polished. On the other hand, it also gave me a clean separation of concerns between the UI, agent loop, and R tooling. And lastly, Go has rich package support for building LLM applications, and its very easy to compile and target any OS.

# Future directions

There's plenty of directions to take Rishi next and continue maintaining it. 

High on the list is expanding model support beyond Anthropic and OpenAI to include Gemini and local models via Ollama. I also want to add more powerful tools to the agent loop like `grep` for searching across files and web search. And although Rishi can execute code in the console, Rishi can't actually read the console output yet. To do this, I'll probably need to make some contributions to the `rstudioapi` package.

On the UX side, I'm exploring ways to make code review more intuitive. Right now, Rishi can make edits, but I think there's room to improve how users review and accept changes before they're applied than just using the Git pane.

Ultimately, I'm just excited to see what people will accomplish with Rishi. If you end up trying it out, I'd love to hear your feedback and ideas for where to take it next.

