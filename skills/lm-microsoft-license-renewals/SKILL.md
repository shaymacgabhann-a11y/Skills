---
name: lm-microsoft-license-renewals
description: Pull every Microsoft license (Microsoft 365, Office 365, Exchange, Teams, Defender, Intune, Entra, Power BI, Visio, Project, Dynamics, Windows 365 and similar) for a client from ScalePad Lifecycle Manager, with seat counts and expiration dates, and turn it into a clean customer-facing renewal summary document. Use this whenever the user asks about Microsoft licensing for a client, license expiration or renewal dates, upcoming Microsoft renewals, M365 seat counts or unused seats, or wants a licensing report, renewal schedule or license summary to share with a customer. Trigger even if the user only names a client and says something like "M365 renewals" or "license report".
---

# Microsoft License Renewal Summary

Build a customer-facing summary of a client's Microsoft licenses and when they expire, using live data from ScalePad Lifecycle Manager. This skill only reads data. It never changes licenses, and it never sends the document to anyone: the user reviews it and shares it themselves.

## Core standard

The document goes in front of a customer, so every date and seat count in it must come from the data you pulled today. Never estimate a renewal date, fill in a missing one, or guess whether a license auto-renews. If the data doesn't say, the document says "Not reported" and the user gets a note about it.

## Calling the ScalePad MCP tools

The ScalePad MCP tools take an operation name and an arguments object. The operation name is the snake_case form of the operation listed in the tool description (for example "Search Clients" becomes `search_clients`), and request parameters go inside `query`:

```json
{
  "operation": "list_saas_assets",
  "arguments": {
    "query": { "filter[client.id]": "eq:<client id>", "page_size": 200 }
  }
}
```

If your client calls the ScalePad REST API directly instead, the same parameters go in the query string of the endpoint named in each step.

## Workflow

### 1. Resolve the client

Search for the client by name: clients query tool, operation `search_clients`, `query: {"search": "<name>"}` (REST: `GET /lifecycle-manager/v1/clients/lookup`). Search is a partial match, so one name often returns several clients. Each result has an `id`, a `label` (the client's name in Lifecycle Manager) and a `license` (the client's Lifecycle Manager license, or null if it has none).

A client with a null `license` can't be read by the licensing endpoint (it returns 403), so take that into account when you pick:

- If exactly one result matches the name the user gave (ignoring case) and it has a license, use it.
- If the exact match has no license but another result does, tell the user and ask which record to use before going on. Licenses are often synced under a different record than the one with the matching name. To make the choice easy, first run the step 2 query with `"page_size": 1` on each licensed candidate (this only reads data) and include each one's `total_count` of SaaS licenses in your question.
- If several results are plausible and none is an exact match, list them with their license status and ask the user which one they mean.
- Use one client record per document. If the user wants licenses from several records in one document, they need to say so.

Capture the client `id` and `label`. If the label looks like a system or internal name rather than how the customer would want to be addressed (for example an all-lowercase name, a run-together name or one with "test" in it), ask the user what name to show on the document.

### 2. Pull the client's licenses

List the client's SaaS assets with the Lifecycle Manager licensing query tool, operation `list_saas_assets`, `query: {"filter[client.id]": "eq:<client id>", "page_size": 200}` (REST: `GET /lifecycle-manager/v1/assets/saas`). Use this Lifecycle Manager tool rather than any Core assets tool that also lists SaaS assets, since this one returns the trial, free and term fields used below.

The maximum `page_size` is 200. If `next_cursor` isn't null, call again with the same query plus `"cursor": "<next_cursor>"`, and repeat until `next_cursor` is null. Missing a page means missing a renewal.

If the call fails:

- **403 Forbidden** usually means the client has no Lifecycle Manager license, or the API key can't see this client. Tell the user and go back to step 1 to choose a different record with them. Don't switch records without asking.
- **No results** means no SaaS licenses are synced for this client. Tell the user to check that the client's Microsoft 365 tenant is connected in ScalePad. Don't produce an empty document.

### 3. Keep the Microsoft licenses

Keep a license when its product name refers to a Microsoft product: Microsoft, Office 365, M365, Exchange, Teams, SharePoint, OneDrive, Defender, Intune, Entra, Azure AD, Power BI, Power Apps, Power Automate, Visio, Project, Dynamics, Windows 365, Copilot, and similar. If you're not sure whether something is a Microsoft product, include it and flag it to the user.

Leave out developer, sandbox and demo subscriptions (for example a product name containing "Developer" or "Sandbox"). They aren't paid renewals the customer needs to plan for. Mention them to the user in step 7, and include them only if the user asks.

If no licenses are left after filtering, don't produce a document. Tell the user what you found and what you left out, and ask whether they want the left-out licenses included.

If the user does include a developer, sandbox or demo subscription, label it in the document as a development or test subscription that isn't a paid renewal. Don't use the trial-conversion or seat-savings wording from the template for it.

If the user asked for all licenses rather than only Microsoft, skip this filter and title the document accordingly.

### 4. Build the license table

For each license, record:

| Field | Source | Notes |
| --- | --- | --- |
| Product | `product.product_name` | Use the name as reported. |
| Status | `status`, `is_trial`, `is_free` | See the status rule below. |
| Seats purchased | `pool.capacity` | "Unlimited" when `is_unlimited_capacity` is true, "Not reported" when null. |
| Seats assigned | `pool.utilized` | |
| Seats unassigned | `pool.available` | If null but capacity and utilized are both reported, use capacity minus utilized. Otherwise leave blank. |
| Term start | `term.starts_at` | Used for the checks in step 5. Not shown in the customer document. |
| Expires | `term.ends_at` | Date only. "Not reported" when null. |
| Auto-renews | `term.is_auto_renewed` | "Yes", "No", or "Not reported" when null. Never assume. |
| Days until expiry | today and `term.ends_at` | Calendar days from today's date to the expiry date, both taken as UTC dates (ignore the time of day). Negative means already expired. |

Status rule: `status` is `enabled`, `suspended` or `unknown`. Show the first of these that applies:

1. "Suspended" when `status` is `suspended`
2. "Trial" when `is_trial` is true. A trial is worth calling out because it stops at term end unless it's converted.
3. "Free" when `is_free` is true
4. "Active" when `status` is `enabled`
5. "Unknown" otherwise

Group each license into a renewal window by days until expiry:

- **Expired**: less than 0
- **Next 30 days**: 0 to 30
- **31 to 90 days**: 31 to 90
- **91 to 180 days**: 91 to 180
- **More than 180 days**: over 180
- **No expiry date reported**: `term.ends_at` is null

Sort the table by expiry date, soonest first, with licenses without a date last.

### 5. Flag things worth the user's attention

Before writing the document, note anything the user should know about or decide on. These go to the user, not into the document, unless the user asks:

- licenses that are expired or expire within 30 days
- trials, and their end dates
- licenses with no expiry date or no auto-renew status
- unassigned seats, which are a cost-saving conversation
- suspended licenses or seats, or seats in a grace-period warning (`pool.suspended`, `pool.grace_period_warning`)
- a term longer than three years (from `term.starts_at` to `term.ends_at`), which can mean the source system is reporting an agreement date rather than the current billing term; suggest the user confirm in the Microsoft admin center or their distributor portal. This flag doesn't change the license's status or how the document describes it.

### 6. Write the customer document

Follow `references/customer-doc-template.md`. Write it in the format the user asks for. If they don't say, use your client's native document format (for example a Word, Google Docs or Claude document), or Markdown if nothing else is available.

Rules for the customer-facing version:

- Write in plain language for a business reader, not an IT admin. Explain what a renewal window means for them.
- Include the client's name, the date the data was pulled, and the MSP's name if the user has given it.
- Leave out internal ids, API field names, integration names, pricing and margin, and anything from step 5 the user hasn't approved for the customer.
- Keep recommendations neutral and factual, such as "review seat count before renewal". Don't make up prices or savings figures.
- Say plainly that the dates come from the Microsoft licensing data synced into the MSP's systems as of the pull date, and that the MSP will confirm before any renewal decision.

### 7. Hand it back for review

Give the user:

- the document (or a link or file path to it)
- a short summary: the number of licenses in the document, the next expiry date, and how many licenses fall in each renewal window. Mention separately any licenses you left out and why.
- the flags from step 5
- anything you couldn't read or weren't sure about, such as missing pages, unclear products or a 403 on a related client

Don't send the document to the customer or anyone else. That's the user's call.

## Portfolio mode

If the user asks for Microsoft renewals across all clients (for example "which Microsoft licenses expire this quarter?"), skip the client filter in step 2, page through every SaaS asset, apply steps 3 to 5, and produce an internal table grouped by client and sorted by expiry. This is an internal report for the MSP, not a customer document, so ids and flags can stay in. Offer to produce customer documents for the clients with the soonest renewals.
