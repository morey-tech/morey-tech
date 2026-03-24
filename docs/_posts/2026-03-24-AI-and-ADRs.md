---
title: "AI and ADRs: A Platform Engineering Love Story"
date: 2026-03-24 00:00:00 -0000
excerpt_separator: "<!--more-->"
categories:
  - Homelab
tags:
  - openshift
  - claude
  - ai-assisted-coding
header:
  image: /assets/images/github-adrs.png
---

For years, I’ve always wanted to use Architecture Decision Records (ADRs) more in the crafting of my demonstrations and in my home lab. When you're operating as a platform engineer or automation engineer in the world of Kubernetes, OpenShift, and Infrastructure as Code, there are a lot of decisions that get made. Documenting the "why" behind those choices is incredibly valuable.

In fact, back in July 2024, I even put the skeleton for a Markdown Architecture Decision Record (MADR) into my home lab repository. And then? I just never did anything with it.

In reality, it was never feasible for me to maintain them. It was way too burdensome—just a bunch of effort to write them out, evaluate all the options, manage the superseding of old records, and document it all while trying to move fast.

But here we are a couple of years later, and tools like Claude Code or Roo Code have completely changed the equation.

### **The OpenShift Software Factory Project**

I’ve been working on a project for about a week now: a simple exercise to deploy a number of operators and operands to manage a fresh, pre-provisioned cluster. (Technically, it’s OpenShift on OpenShift, but to me, it just looks like a normal OpenShift cluster.)

I wanted to use Argo CD as much as possible and follow GitOps paradigms, while using Ansible for the initial bootstrapping. Why? Because *something* has to actually get the GitOps operator deployed to the cluster so it can respond to the applications you create. After that, Argo CD owns everything, and everything is managed with Kubernetes as the source of truth.

When putting together this demo repository, a lot of choices had to be made. But this time, I used AI to help me build the documentation alongside the code.

As the orchestrator, I get to determine what gets worked on and the approaches we take, while prompting Claude to review the changes made and document any pertinent architecture decisions in markdown. In just a week, I already have over 30 ADRs.

### **The AI \+ ADR Feedback Loop**

These records have actually come really in handy because they've helped establish patterns I can refer Claude back to.

Of course, you could already do that to some extent. You could point Claude at an existing YAML manifest, a playbook, or a previous example to show it *how* you did something. That's important. But the added documentation of an ADR provides the context on *why* those choices were made.

It creates a history. Already in this project, I've had previous ADRs superseded by new ones that address new capabilities or requirements that didn't exist when the original choices were made.

Now, I can reference a past architecture decision and say, "I want you to implement X again, following the pattern documented here." Claude has access to the code from last time, but it *also* has the broader context of why that approach was taken, so it can avoid making the same mistake again.

### **A Real-World Example: Leaving Room for Imperativeness**

Let me give you a concrete example of where this shines. Part of the goal of this project is portability. I should be able to take virtually any OpenShift cluster that meets the prerequisites—be it on-prem bare metal, managed in the cloud like Azure Red Hat OpenShift, or even OpenShift Local—and run this automation.

But I hit a snag: I had originally hardcoded the apps domain for my cluster in a Kustomize component stored in the Git repository. Because of that, I couldn't just simply run the same bootstrapping against a different cluster.

So, alongside Claude Code, we came up with a solution to discover the apps domain at runtime and patch the GitLab instance. We established a pattern using a Kubernetes Job and Argo CD sync waves. Here’s how it works: on sync wave 0, the GitLab custom resource (CR) gets deployed to the cluster. On sync wave 1, a Job deploys that patches the GitLab CR with the cluster's specific apps domain.

Now, granted, mutating state outside of Git is a bit of a GitOps anti-pattern. But at the very least, the Job and its logic are managed through GitOps, so I think it's a fair compromise.

This also required us to practice a concept explained in the Argo CD documentation: [*leaving room for imperativeness*](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/#leaving-room-for-imperativeness). By explicitly omitting the host domain value from the custom resource in our Git repository, we let the Job inject it later. This brilliant little trick circumvents Argo CD constantly seeing an out-of-sync diff on that resource.

All of that logic, the rationale, the compromises, and the documentation that it supersedes the old Kustomize approach is completely captured in [**ADR-0022: Runtime Apps Domain Discovery for GitLab**](https://github.com/morey-tech/openshift-software-factory/blob/main/docs/decisions/0022-runtime-apps-domain-discovery-for-gitlab.md).

Since writing that ADR, I've been able to easily replicate that exact same pattern multiple times for other areas in the bootstrapping process just by pointing the AI back to it.

### **Looking Forward**

It is incredibly easy to prompt AI to do the heavy lifting of writing the code and maintaining the documentation, while you guide the architecture. It took something that was practically unfeasible for me to manage alone and turned it into a seamless part of my workflow.

Frankly, I just find it fascinating, and I'm excited to see what a few years of operating like this accomplishes.

*Curious to see what 30+ AI-assisted ADRs look like in practice? Check out the demo repository here:* [**openshift-software-factory**](https://github.com/morey-tech/openshift-software-factory)