---
layout: post
title: "How I built MCP servers for systems that have no API"
date: 2026-05-04
description: "Build MCP servers for legacy systems without APIs. Parse HTML, validate schema, ship fast."
---

## The moment you realize there's no API

It's 2 PM on a Tuesday. I've got Claude in one window, a 15-year-old enterprise system in another. Marketing needs that system integrated into their LLM workflows. They're not asking for much—just pull the customer data, validate against schema, feed it back. Except the system has no API. Never did. The vendor charges for API access like it's 1998. So you have two choices: pay them, or accept a different boundary. I picked the third option: wrap the cookie-authenticated session, treat the HTML as unstable, and build the stability layer on top. By Wednesday afternoon, the LLM had native access to live data. By Thursday, it was in production.

This is how you actually solve that problem.

## Why MCP is still the right abstraction

You might think: if there's no API, maybe MCP doesn't matter. Skip it. Query the system directly from your Claude invocation.

Don't.

MCP gives you three non-negotiable things, even without an API. First: *schema validation at the boundary*. You define what your tools accept and return. The LLM sees that schema. You enforce it before touching the fragile system. Second: *a stable integration point*. Your code breaks, you rewrite the internals—the MCP layer stays the same. The Claude invocation stays the same. You're not touching your orchestration every time the target changes. Third: *composability*. You wrap one system today. Next week you wrap another. By month two you have a unified namespace across five legacy systems, and Claude navigates them like they're one coherent thing.

An unauthenticated web scraper gives you none of that. MCP + a thin scraping layer gives you all three. The abstraction is still valuable.

## Architecture: three layers

Here's how to think about it. Three layers, stacked.

**Layer 1: The cookie jar.** This is authentication. The target system uses cookie-based sessions. You log in once—or once per day, or once per week, depending on session TTL—and get back a session cookie. You store it. File, Redis, environment variable, wherever. On each tool invocation, you load it. If it's stale, you refresh it. Dead simple, and it's the critical dependency. Everything downstream depends on this being fresh.

**Layer 2: The fetch client.** This is your translation layer. The MCP tool doesn't talk HTML. It doesn't build form-encoded requests. The fetch client does that. It takes your validated MCP inputs—JSON, typed, bounded—and converts them to the weird, hyperlink-fragmented requests the target system expects. It fires them off with the cookie. It reads back HTML. It parses the HTML using selectors or lightweight regex. It returns structured data: JSON, with guaranteed shape.

**Layer 3: The tool layer.** This is your stable interface. It defines schemas, calls the fetch client, validates responses, returns MCP tool results. This layer never touches HTML. It never thinks about cookies. It speaks JSON in, JSON out. The tool layer is what Claude sees. It's what you version. It's what you test.

Data flows downward through layers 3 → 2 → 1, outbound to the target. Responses flow upward: 1 → 2 → 3, back to Claude. If the target HTML changes, you rewrite layer 2. Layers 1 and 3 barely move. The tool layer stays stable even though the boundary is fragile. That's the architecture.

## The fragility tax

I'm not going to pretend this is risk-free. The target system owns its own HTML. When it upgrades, when it changes CSS selectors, when it adds a CSRF token, your selectors break. Your regex fails. Your response shape validation fires and you get an alert at 11 PM on a Friday.

This is the fragility tax. You pay it.

How to detect breakage: *schema-drift errors*. Your fetch client parses the response and tries to shape it into your known structure. You're missing a field. The field moved. The field is now nested three levels deeper. Your validation fails loudly. You log it. You alert on it. You don't silently return garbage to Claude. *Response shape validation* at the fetch layer—before you hand anything to the MCP tool—is your first defense.

How to recover: *fail fast*. When validation fails, your MCP tool returns an error. Claude sees it. The user sees it. You don't return stale data or make something up. You also need *manual override*. Your cookie expires or the session breaks. You have a script or a process to refresh it without deploying code. You set an environment variable. You rebuild the container. You get live again in five minutes, not two hours. And *scheduled re-validation*. Once a day, you make a canary request to the target. Parse the response. Validate shape. If it fails, you alert before users hit it. You find out at 8 AM, not at noon during a critical prompt.

Is this more maintenance than hitting a real API? Yes. You take the tax in exchange for capability you literally cannot get otherwise. That's the honest framing.

## Worked example: a single tool in pseudocode

Let's say you're pulling customer contact info. One tool: `get_customer_contact`. Input: a customer ID. Output: name, email, phone.

```
tool get_customer_contact(customerId: string) -> ContactInfo:

  // Layer 3: Validate input
  if not customerId matches /^[A-Z0-9]{6,10}$/:
    return error("Invalid customer ID format")

  // Layer 3 → Layer 2
  response = fetch_client.get_customer(customerId)

  // Layer 2: Parse response
  html = response.body

  // Defensive: is the HTML what we expect?
  if not html.includes("Customer Contact"):
    log_alert("Response shape changed, selector not found")
    return error("Integration broken, manual review needed")

  // Parse the HTML
  name = html.select(".customer-name").text()
  email = html.select(".customer-email").text()
  phone = html.select(".customer-phone").text()

  // Validate structure
  if not email matches /@/:
    log_alert("Email field malformed or missing")
    return error("Data validation failed")

  // Layer 2 → Layer 3
  contact = ContactInfo {
    name: name,
    email: email,
    phone: phone
  }

  // Layer 3: Type-check and return
  if not contact.validate():
    return error("Output schema violated")

  return contact
```

Three layers, visible. Input validation. Fetch. Parse. Canary checks at each step. If the target breaks, you catch it at the parse layer, alert, return an error. The MCP layer—the part Claude sees—never lies. It either returns a valid contact or an error.

## When NOT to do this

Three cases where you stop and find another path.

**The target has an API.** Check hard. Does it have a REST API? A GraphQL endpoint? An official SDK? Use it. Officially-supported APIs have documentation, rate limits you can plan for, change processes you can read. They're not fragile in the same way. Do the simple thing that works: hit the real API.

**The Terms of Service prohibit scraping.** Read them. If they say no automated access, no bots, no scraping—honor it. You might believe you could scrape anyway. You might be right. But you're choosing to break a contract the organization signed. Find the organization's security team. Ask them about integrations. Usually there's a path that's blessed.

**Anti-bot and rate limits make it impractical.** Some systems have CAPTCHA, browser fingerprinting, or rate limits so aggressive that you can't make 10 requests a day. If you're seeing 429s after three requests, you're not going to build a reliable integration. Look for another path: the vendor's support team, a data export process, an unofficial-but-stable import, a webhook from a third-party service.

Those three? You stop and pivot. Otherwise, you build the layer.

## Reference implementation

I've open-sourced a reference implementation at [`github.com/addiplus/mcp-cookie-auth-reference`](https://github.com/addiplus/mcp-cookie-auth-reference). It's a complete, production-pattern example: a Hono mock target that mimics a legacy system with cookies and HTML forms. A full TypeScript MCP server wrapping it. Tests for the three-layer pattern. Examples of cookie refresh, response validation, error handling. If you're building this, start there. Copy the structure. Adapt the selectors.

## What this looks like in 2026

The dream of "api-first" enterprise software was always going to hit reality. Reality wins. Most systems you need to integrate were built before APIs mattered. They'll be running for another five years minimum. Scraping them is not elegant. It is, however, tractable.

An MCP server plus a fetch-and-parse client plus decent schema validation turns "we don't have an API for that" from a blocker into a two-day project. You log in once. You define the shapes. You build the tools. By Thursday afternoon, Claude has live access to data that had no programmatic path last week.

That's what shipping looks like now.

---

*Joe Smith — AI-native engineer in Seattle. Builds MCP servers, Claude skills, and recursive sub-agent swarms on top of Claude Code. Public work at [github.com/addiplus](https://github.com/addiplus).*
