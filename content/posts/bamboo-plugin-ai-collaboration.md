---
title: "When Two AIs Argued Their Way to a Better Bamboo Plugin"
date: 2026-06-02
author: "Claudio Lombardo"
tags: ["AI", "DevOps", "Bamboo", "Artifactory", "Java", "Open Source", "UCSD"]
categories: ["Engineering", "AI Collaboration"]
description: "JFrog abandoned the Bamboo Artifactory Plugin. Bamboo 9.6 was reaching end of life. Rewriting hundreds of pipelines was not an option. Here's how two AI systems and one engineer fixed it."
draft: false
---

## The Problem Nobody Wanted

Early in 2026, our team found itself staring at a problem that nobody was particularly excited to own.

Bamboo 9.6 — the CI/CD platform used by engineering teams across UC San Diego — was approaching end of life. Atlassian's recommended path was clear: upgrade to Bamboo 12.1 Data Center.

Simple enough.

Except for one small detail.

The JFrog Artifactory Plugin, used by virtually every team to publish build artifacts, did not work on Bamboo 12.1.

Even better, JFrog had quietly stopped maintaining the plugin over a year earlier.

The options were ugly:

- Rewrite every Bamboo plan by hand.
- Find a commercial replacement.
- Stay on an unsupported platform and hope nothing bad happened.
- Explain to hundreds of developers why their pipelines suddenly stopped working.

None of those sounded particularly fun.

So I picked a different option.

## Putting Two AIs in the Same Room

I've been watching AI coding assistants evolve for a while, and I had a theory.

Everybody was talking about replacing engineers with AI.

I was more interested in making AI systems review each other.

Different AI models are good at different things. More importantly, they fail in different ways. One model will confidently walk straight into a wall while another immediately spots the problem.

That sounded suspiciously similar to peer review.

So instead of asking one AI for help, I recruited two.

- **Claude** from Anthropic
- **Summer** from OpenAI Codex

My role wasn't to sit back and watch them write code.

My role was to direct the investigation, challenge assumptions, compare solutions, and decide what was actually safe to ship into production.

Think less "AI replacing engineers" and more "AI-powered engineering team with an opinionated project manager."

## The Migration Begins

The plugin had been built for a completely different world.

It targeted Bamboo 9.x, Java 11, older Atlassian frameworks, older Struts versions, and a dependency ecosystem that no longer existed.

Bamboo 12.1 had moved on:

- Java 21
- Struts2 7.x
- Jakarta EE 10
- Updated OSGi bundle resolution
- New security controls

The plugin didn't simply fail.

It failed at multiple layers simultaneously.

Exactly the kind of project everybody loves inheriting.

## Phase 1 — Making It Compile Again

The first round of failures was straightforward, at least by Java standards.

Packages had moved.

Classes had disappeared.

Dependencies had changed names.

`javax.servlet` became `jakarta.servlet`.

Several Struts APIs had changed.

An old `commons-configuration` dependency was triggering runtime failures despite looking harmless during compilation.

Claude worked methodically through the compiler issues.

Summer independently reviewed the fixes and caught dependency-scoping problems that would have created headaches later.

After several rounds:

**Build successful.**

Progress.

## Phase 2 — Making It Load

Compiling is easy.

Running is where the fun starts.

The Bamboo OSGi framework immediately began complaining about missing packages.

One package in particular:

```text
com.opensymphony.module.propertyset
```

Bamboo 12.1 no longer exported it.

Without it, the plugin would never start.

Claude identified the missing dependency and proposed bundling it directly within the plugin.

A perfectly valid OSGi approach.

Summer independently verified the solution and noticed something subtle:

The exported version string had to match exactly what Bamboo expected:

```text
1.6.0.atlassian_8
```

Not close.

Not approximately.

Exactly.

One wrong version string would have produced silent bundle-resolution failures that are the software equivalent of stepping on a Lego in the dark.

After that adjustment:

**Plugin enabled successfully.**

## Phase 3 — The HTTP 500 That Refused to Die

This is where things became interesting.

The plugin installed.

The plugin enabled.

The task configuration screens loaded.

Everything looked good.

Then we clicked **Save**.

HTTP 500.

Every time.

No meaningful error.

No obvious stack trace.

Just failure.

I started tracing requests through Bamboo's internals.

Claude dug into the interceptor chain and followed the problem all the way into:

```text
ReadOnlyConfigurationEditInterceptor
```

Eventually we discovered Bamboo couldn't find the plan information during task save operations.

The plan key was missing.

Claude's diagnosis was solid.

Struts2 7.x strict parameter handling was rejecting form fields containing dots in their names.

The proposed solution:

- Rename roughly seventy fields
- Update multiple templates
- Add translation logic
- Maintain backward compatibility mapping

It would have worked.

But it was invasive.

And risky.

So before touching seventy fields, I asked Summer to independently investigate the same issue.

Summer came back with a completely different answer.

The form fields weren't the problem.

The plan key simply wasn't being submitted.

Instead of redesigning the forms, Summer proposed injecting the missing value during submission using a lightweight helper.

The result was a small JavaScript helper:

```text
ensureTaskPlanKey.ftl
```

The helper retrieves the plan key from multiple sources:

- FreeMarker context
- Form action URL
- Current page URL
- HTTP referrer

Then injects the hidden fields before submission.

No field renames.

No Java changes.

No migration risks.

Problem solved.

Here's the funny part:

Claude wasn't wrong.

Summer wasn't magically smarter.

Both solutions would have worked.

But one required surgery.

The other required a band-aid.

Without two independent perspectives, we almost certainly would have implemented the more complicated solution.

That's the moment where the entire experiment paid for itself.

## What I Actually Did

I want to be clear about something.

I didn't sit back while two AIs magically fixed a plugin.

I also didn't personally write thousands of lines of Java.

What I did was direct the engineering effort.

When the HTTP 500 appeared, I pushed for root-cause analysis instead of guessing.

When one solution looked expensive, I asked for a second opinion.

When two valid solutions emerged, I evaluated risk, maintainability, compatibility, and operational impact.

Engineering isn't typing.

Engineering is deciding.

The hard part wasn't writing Java.

The hard part was recognizing when we were about to spend a week implementing the wrong solution.

That's still a human job.

## The Result

The final result exceeded expectations.

The **UCSD Bamboo 12.1 Artifactory Plugin** now runs successfully on Bamboo 12.1.7.

![UCSD Bamboo Artifactory Plugin running on Bamboo 12.1](/images/bamboo-plugin-12.1-running.png)

*The Bamboo 12.1-compatible Artifactory Plugin running successfully with all 34 modules enabled.*

Existing plans continue to work without reconfiguration.

No mass migration.

No rebuilding hundreds of pipelines.

No emergency project consuming months of engineering time.

According to Sven and Long, avoiding manual plan rewrites saved hundreds of engineering hours for their team alone, not counting every other Bamboo user across the organization.

The project passed security review and is now being used in production.

## Open Source

The plugin remains open source under the original Apache 2.0 license.

Source code:

[ucsd/bamboo-12.1-LTS-artifactory-plugin](https://github.com/ucsd/bamboo-12.1-LTS-artifactory-plugin/)

Project documentation and downloads:

https://ucsd.github.io/bamboo-12.1-LTS-artifactory-plugin/

If your organization is facing the same Bamboo 9.6 end-of-life challenge, hopefully this saves you from solving the same problem twice.

## The Bigger Lesson

The biggest lesson from this project isn't that AI can write code.

We already know that.

The lesson is that independent AI systems reviewing each other produce better engineering outcomes than a single AI working alone.

That's not revolutionary.

We've been doing the same thing with humans for decades.

Peer review.

Code review.

Red teams.

Independent validation.

The difference is that now those additional perspectives can arrive in seconds instead of days.

The engineer's role doesn't disappear.

If anything, it becomes more important.

Someone still has to define the problem, challenge assumptions, evaluate competing solutions, manage risk, and decide what gets deployed into production.

The tools changed.

The responsibility didn't.

And honestly?

Watching two AIs argue over a Java plugin while Bamboo threatened to ruin everyone's week was far more entertaining than rewriting hundreds of pipelines by hand.

I'll call that a win.

---

*Claudio Lombardo is a Principal IT Infrastructure Engineer at UC San Diego. This project was completed using a collaborative engineering approach involving Claude (Anthropic), Summer (OpenAI Codex), and a stubborn refusal to rewrite hundreds of Bamboo plans.*
