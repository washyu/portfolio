---
title: "My Approach to Test Automation Strategy"
date: 2026-01-05
draft: true
---

After 20 years in test automation at companies like Amazon, Microsoft, and Blizzard, 
I've developed a straightforward approach that works across most projects.

## Start With What Exists

Before writing any new tests, I look at what developers have already done:
- What's the current unit test coverage?
- What integration tests exist?
- Where are the gaps?

## Make It Visible

Dashboards are underrated. You can't improve what you can't see. I set up coverage 
dashboards early so the whole team knows where we stand.

## E2E Tests Should Cover the Gaps

End-to-end tests are expensive to maintain, so I focus them on:
- Integration points that unit tests can't reach
- User stories from actual documentation
- Critical paths that would break the business

## Keep It Simple

The best test framework is one that other people can actually use. If it needs a 
manual to run, it's too complicated.
