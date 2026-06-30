---
layout: post
title: "Please stop flooding our projects with AI slop to furnish your CV"
date: 2026-06-30 22:23:14 +0100
author: Neil
index: true
---

Successful contributions to open source projects are a kind of currency. GitHub in particular encourages this in a number of ways: by showing avatars of contributors on repository pages, by showing your contributions to your followers via the activity feed and by signalling contributions per day on the activity graph of your profile. Potential hiring managers often take note of this. Recruiters often find and screen candidates this way. If you are a software developer (either existing or aspiring) looking for work, tuning these signals can often work to your advantage.

As an open source maintainer, it’s quite noticeable how the pattern of external contributions has changed in the last year. We’re far more likely to receive pull requests instead of issues. If we do receive issues, they often come with an AI-generated analysis attached. We’re receiving far more security vulnerability reports than ever before and often they even come with AI-generated fix proposals attached too.

I don’t doubt that some of these contributions are from people who are genuinely interested in what we do, but the cynical part of me believes that a substantial amount of this is that people are realising that AI can be used to game GitHub to their own benefit. It’s now easy to ask Claude to generate a list of interesting open source projects, then ask Claude to find some problems in them, and then ask Claude to raise some PRs to fix them. You don’t even have to use the projects or care about them, but you can easily create the illusion to outsiders that you care, or that you found a problem, or that you put the time into fixing it. On the internet, nobody knows you’re a dog, but with the help of LLMs, you can effortlessly overstate your human abilities on your GitHub profile.

Recently, a contributor with virtually no GitHub-wide contributions from late 2018 up until a couple weeks ago, with no prior engagement with our project that we know of, raised three separate PRs to correct spelling and grammar mistakes in comments. Claude made the fixes, presumably wrote the PR descriptions, even signed off the commits on behalf of the user and then helpfully inserted its co-authorship into the commit message trailers. Maybe it even opened the PRs itself, who knows. I’d be fascinated to know whether the prompt was to “go and find issues” or whether to focus on spelling and grammar issues in particular for whatever reason.

The changes were harmless and correct, but that did not make me feel better about accepting or merging them. Instead I couldn’t help but ask myself: why this, why now? Why, out of all of the issues and `TODO`s and `FIXME`s in our codebase are they submitting *this*? And then it dawned on me that these contributions weren’t about our project at all.

I closed all three PRs without comment.

Maybe this was unreasonable, but truthfully, I’m just not interested in encouraging people to take up our time with this kind of busywork. I do not want to set a precedent of accepting PRs that materially improve nothing, nor do I want our contributor list to become a reward for asking a robot to fix typos.

The same pattern has emerged with security vulnerability reports. CVEs traditionally are credited to their reporters, but all of the reports that we have received recently have been obviously AI-generated. Security fixes are always important of course, but again I find myself wondering if this is happening because people care about the fixes or because they are looking for an easy credit. We have been far more selective lately when evaluating the severity of such reports and, in some cases, declining to issue CVE notices for low-severity items. I have some feelings about the fact that private disclosure is dying anyway, which I may write about another time, but the effort involved in coordinating private fixes and disclosure notices and releases is substantial enough to *require* us to be selective.

Ultimately, open source is built on trust. The metric that matters is not how many pull requests you can persuade an LLM to produce, nor how many CVEs you can accumulate, but whether you can make a project meaningfully better. If you want to contribute to open source projects, contribute because you care. If all you want is another green square or another contributor badge, please go elsewhere.
