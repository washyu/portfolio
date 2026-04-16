---
title: "Testing an MCP Server with TestSprite: Bridging the Protocol Gap"
date: 2026-04-15
draft: false
tags: ["testing", "ai", "mcp", "homelab", "hackathon", "fastapi", "python", "claude"]
description: "I entered the TestSprite hackathon to AI-test my MCP server. There was just one problem: TestSprite tests REST APIs."
---

I entered the TestSprite Season 2 hackathon with a straightforward goal: use their AI-powered testing agent to validate [homelab_mcp](https://github.com/washyu/homelab_mcp), my open-source MCP server for homelab infrastructure management. What I didn't anticipate was that the most interesting engineering challenge wouldn't come from the server itself. It came from the protocol mismatch between where TestSprite lives and where MCP servers live.

## What is homelab_mcp?

homelab_mcp is an MCP (Model Context Protocol) server that exposes homelab infrastructure operations as tools that AI assistants like Claude can invoke. It covers Proxmox VM management, SSH execution, Ansible playbook runs, Terraform drift detection, and a handful of other things you'd want to automate in a self-hosted environment. It's published on PyPI and distributed via `uvx`, indexed across several MCP community directories.

The tools talk to real infrastructure: Proxmox nodes, TrueNAS, Active Directory, remote hosts over SSH. That last part becomes important later.

## The Problem: MCP ≠ REST

TestSprite is built to test REST APIs. You point it at a Swagger/OpenAPI spec, it bootstraps a test plan, and its AI agent runs test cases against your endpoints. That's a great model, but MCP servers don't speak HTTP REST. They communicate over stdio using the Model Context Protocol, which has its own handshake, message framing, and tool invocation format.

Out of the box, TestSprite has no way to talk to an MCP server directly. I found this out the hard way during a pre-hackathon experiment where I got a 40% pass rate, and all the failures traced back to test harness issues, not server defects. Missing the MCP `initialize` handshake, wrong `Accept` headers, a missing `websockets` module in TestSprite's sandbox. Zero actual bugs in the server itself.

That was actually a useful finding. It meant the approach needed to change.

## The Solution: A FastAPI Bridge

The fix was conceptually simple: wrap homelab_mcp in a FastAPI layer that exposes each MCP tool as a proper REST endpoint. TestSprite can hit REST. So give it REST.

```
TestSprite Agent → FastAPI wrapper → homelab_mcp tools → Infrastructure
```

Each tool gets a POST endpoint. Request bodies map to tool parameters. Responses come back as JSON. Standard stuff, but with one non-trivial design decision baked in.

## Getting the Error Semantics Right

Many of homelab_mcp's tools talk to external resources: Proxmox APIs, SSH targets, TrueNAS, Active Directory. If any of those are unreachable, the tool can't succeed, and that failure isn't the API's fault. It's a dependency failure.

The correct HTTP status for that is **424 Failed Dependency**, not 500. Returning 500 would imply something went wrong in *my* code. Returning 424 correctly communicates "everything on my end is fine, but a resource I depend on isn't cooperating."

I updated the PRD and Swagger spec to document 424 as a valid response for all tools that touch external resources, with a full payload contract: `status=failed_dependency`, `host`, `port`, `protocol`, and a `requires` remediation hint.

## The Five-Run Journey to 10/10

Here's the full arc, because the messy reality is more useful than the clean summary:

| Run | Pass Rate | What Happened |
|-----|-----------|---------------|
| 1 | 7/10 (70%) | 424 contract validated end-to-end; 3 real bugs surfaced |
| 2 | 9/10 (90%) | Fixed validation bugs; 1 remaining failure was a doc inaccuracy |
| 3 | 7/10 (70%) | Regenerated test plan was stricter, exposed new gaps |
| 4 | 8/10 (80%) | ResourceManager wiring fixed; 2 wording mismatches remain |
| 5 | **10/10 (100%)** | Schema and error message fixes close everything out |

### Run 1: 70% - Real Bugs Found

After the FastAPI wrapper and 424 documentation, the first run immediately validated the preflight design. All 5 external dependency tests passed, correctly asserting the full 424 payload shape. But it also surfaced three legitimate bugs:

- **Missing Pydantic validation:** `_register_tool_route` used raw `request.json()` with no model bound, so missing required fields crashed handlers and returned 500 instead of 422. This affected all 56 tools.
- **Wrong field name in code summary:** `bulk_discover_and_map` was documented as taking `hosts` but the actual schema required `targets`. TestSprite generated a test against the wrong contract.
- **No typed 412 path:** `deploy_vm` had no clean "device not found" error path; unrecognized devices produced generic 500s.

**Key insight:** TestSprite read the Swagger spec and automatically generated negative-path tests for the 424 scenarios. Keep your API docs accurate and the AI agent generates tests that match your actual contract, including failure modes.

### Run 2: 90% - Validation Fixed

Added a `jsonschema` validator compiled once per route at registration time, running before the handler on every request. One change, closed two test failures, and neutralized an entire class of future handler crashes. The remaining failure was the `hosts` vs `targets` field name, a doc issue, not a server bug.

### Runs 3-5: Chasing 100%

Correcting `code_summary.yaml` and regenerating the test plan produced *stricter* tests that exposed new gaps: `ResourceManager` not being initialized in the FastAPI lifespan, error message wording that didn't match TestSprite's substring assertions, and an empty array edge case not covered by `minItems`.

Each fix was small and targeted. The path to 10/10 was iterative, not heroic.

## What I Learned

**1. MCP servers need a REST facade for conventional API testing tools.**
If you want to use TestSprite, Postman, or any HTTP-based test harness against an MCP server, you need a translation layer. FastAPI makes that layer lightweight enough that it's not a burden, and it has the side benefit of making your tools accessible to non-MCP clients.

**2. API documentation is part of your test suite.**
The 424 story is the clearest illustration of this. The moment I put the error semantics into the Swagger spec, TestSprite generated tests for them. The spec isn't just documentation for humans. It's instructions for the test agent. Garbage in, garbage out. Accurate spec in, accurate tests out.

**3. `code_summary.yaml` accuracy matters as much as the spec.**
TestSprite uses your code summary to understand your intent. When the summary said `hosts` instead of `targets`, TestSprite generated a test against the wrong contract. Keeping the summary in lockstep with your actual schemas is essential. Inaccurate docs produce inaccurate tests.

**4. Validation at the framework boundary is free and compounds.**
One jsonschema validator per route closed two unrelated test failures and protected all 56 tools in one change. Push validation as early as possible, before handlers even run.

**5. Error message wording matters more than you'd expect.**
TestSprite asserts on literal substrings. "Device with ID 9999999 not found in sitemap" doesn't match an assertion looking for "device not found". Use consistent, predictable phrasing in your error messages and make sure it aligns with both your classifier patterns and your documented contract.

**6. Set up CLAUDE.md guardrails before letting Claude Code near TestSprite.**
I used Claude Code as my coding agent throughout the hackathon. Without explicit rules, Claude Code will autonomously loop fix→rerun→fix→rerun trying to reach 100%, burning through TestSprite credits fast. I lost intermediate run summaries to exactly this loop. Add these rules to your `CLAUDE.md` before you start:

```markdown
## TestSprite Rules
- DO NOT run TestSprite unless explicitly told to do so
- DO NOT modify any files in testsprite_tests/
- DO NOT modify testsprite_backend_test_plan.json
- Fix failures by correcting SERVER code or code_summary.yaml only
- After each run, copy raw_report.md to raw_report-runN.md before the next run
```

Trigger runs yourself. Commit the report after each run. Then hand control back to Claude Code for the next round of fixes. Manual checkpoints between runs save both credits and history.

**7. Auth and no-auth modes require separate TestSprite runs.**
If your API supports both authenticated and unauthenticated modes, you can't cover both in a single TestSprite session. It targets one server configuration per run. Plan for two separate runs if full auth coverage matters.

**8. Commit your reports after every run, or copy them manually.**
The TestSprite summary report overwrites the previous run's output locally. The web UI shows individual test results but not the full run summary with analysis. Either `git commit` after each run or copy `raw_report.md` to `raw_report-runN.md` before triggering the next run. Otherwise you lose the improvement arc that makes the story (and the hackathon submission) compelling.

**9. TestSprite is only as aggressive as its target allows.**
Without live infrastructure, TestSprite could only exercise the "unreachable" paths (424) and input validation (422). With a real Proxmox target it would likely catch things like invalid node names being silently accepted, duplicate vmids, and out-of-range values, exactly the kind of negative-path testing it excels at generating from the PRD. The limiting factor wasn't TestSprite, it was the test environment.

## The Results

Five files changed in the server to go from 40% to 100%:

| File | Change |
|------|--------|
| `openapi_app.py` | Per-route jsonschema validator; 422 with field path; FastAPI lifespan for ResourceManager; extended error classifier |
| `drift_handlers.py` | Short-circuit to 412 when no baselines registered |
| `network_tools_schema.py` | `minItems: 1` on `bulk_discover_and_map.targets` |
| `vm_operations.py` | Consistent "Device not found" error wording |
| `code_summary.yaml` | Corrected field names; documented 412/422/424 contracts per tool |

TestSprite exercised all four documented error contracts (200, 422, 412, and 424) and validated the full payload shape for each. Every preflight-aware tool was confirmed to short-circuit correctly before touching real infrastructure. That's genuine confidence, not just green checkmarks.

## The Code

The FastAPI wrapper, test suite, and full run history are on the [`TestSprite_Hackathon` branch on GitHub](https://github.com/washyu/homelab_mcp/tree/TestSprite_Hackathon). The stable release of homelab_mcp is on `main` and installable via `uvx` or `pip`:

```bash
uvx homelab_mcp
```

If you're running your own homelab and want Claude (or any MCP-compatible client) to be able to manage it, give it a look.

---

*This post is part of my TestSprite Season 2 hackathon submission. TestSprite is an AI-powered API testing platform. If you're building REST APIs and want autonomous test generation, it's worth checking out.*