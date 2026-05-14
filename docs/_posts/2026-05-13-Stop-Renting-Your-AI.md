---
title: "Stop Renting Your AI: The Strategic Case for Self-Hosted Models"
date: 2026-05-13 00:00:00 -0000
excerpt_separator: "<!--more-->"
categories:
  - Rants
tags:
  - ai
  - business
header:
  image: /assets/images/stop-renting-your-ai.png
---

Lately, I've been thinking a lot about privately hosted AI models—both large language models and smaller, purpose-built ones.

Right now, the default option for most of us is to use Software-as-a-Service (SaaS) AI models. We rely on the major players: Claude from Anthropic, Gemini from Google, or the solutions provided by Microsoft and OpenAI.

But there is a major challenge looming on the horizon: **almost all of these SaaS AI solutions are currently being provided as loss leaders.**

These companies are investing billions of dollars into developing frontier models and the SaaS platforms that give you API access to them. Right now, they are actively providing these services at or below the cost to operate them. They are doing this to drive adoption, get people hooked on their services, and build a massive moat. Ultimately, the goal is to lock consumers into their specific AI ecosystem.

But eventually, the day of reckoning is going to come. These corporations are beholden to their shareholders and are legally obligated to produce a profit. They simply cannot continue this loss-leader model forever.

Once they capture a sufficient user base, they are going to start increasing the cost of these solutions until they become profitable—regardless of whether that new cost exceeds the value their users are actually getting from it.

Of course, there is the argument that they can leverage economies of scale. If they capture enough market share, perhaps they won't have to increase costs significantly. And yes, there are peaks and troughs to the demand curve. If you corner the North American market and expand into the APAC region, the demand curve shifts, and in theory, you can fit more users into the same fixed capacity. There are also technologies that provide intelligent, context-aware routing of requests to make platforms more efficient.

But the reality is, that likely won't be enough.

AI is one of those technologies where it is incredibly hard to squeeze out more efficiency as you scale. Each new user directly increases the demand on the system, requiring additional compute capacity. That wouldn't be an issue if these providers had massive excess capacity sitting around, but nobody does. They are sprinting full speed ahead just to build out the capacity to meet *existing* demand. We aren't going to reach a point anytime soon where bringing on a new user doesn't force a provider to increase their costs.

And they will pass those costs on to you.

This is exactly why I advocate for having the infrastructure and the software to meet your AI demands with your own resources—or at the very least, building off a platform that is interoperable enough that you aren't tied to one specific vendor.

This is where open-source models and projects like OpenShift AI have a huge impact. Open-source models are continuing to be highly competitive with frontier models, and more importantly, they let you pick *whatever hardware you want* to run them on.

If you spin up an AI platform today leveraging GPUs provided by Azure, there is nothing stopping you in the future from purchasing your own GPUs, migrating that platform to your own data center, and operating it at a relatively fixed cost over the lifespan of that hardware.

Of course, going this route—spinning up your own platform—comes with a trade-off.

It requires upfront effort, and it requires the right skill set in-house to make it work. Organizations have to weigh this investment against the risk of sticking with SaaS. And to be clear, I don't just mean the standard, incremental cost increases due to inflation that every business plans for. I mean *significant* price hikes as these AI providers scramble to reach profitability in their delivery of these services.

By choosing to leverage open-source and open-standard solutions to host your own models, you are making a conscious choice to invest in your own organization. You are investing in your internal talent, your maturity, and your ability to deliver these services in-house.

Crucially, it also allows you to craft solutions that are hyper-domain specific to address the exact challenges your organization faces.

Is there a learning curve? Absolutely. But it is not exceedingly difficult.

If your organization is already hosting a Kubernetes cluster or you have a few data scientists on staff, you can get started fairly quickly. You build that initial footprint, and then you scale your maturity over time.

That in-house maturity is what allows you to make informed decisions. It empowers you to pick the right AI solutions and find the actual, practical opportunities to integrate a large language model into your workflows and operations.

The alternative is remaining ignorant to the realities of the technology—and inevitably getting sold on an AI-based solution that doesn't actually help you deliver value to your end users or achieve your real outcomes.

Instead of waiting for your SaaS provider to hike their prices, invest in your own capabilities and take control of your AI infrastructure. Don't get locked in.