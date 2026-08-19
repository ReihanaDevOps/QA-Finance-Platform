Yes. Before building the full QA Finance application, I strongly recommend building the **local AI QA discovery engine as a standalone proof of concept**.

The goal of Phase 0 should be very simple:

> **Give the agent a URL + username + password → let it explore the real application → produce a reliable Functional / Performance / Security scenario list.**

Don't build the dashboard, PostgreSQL, cloud deployment, Test Lab, reports, etc. yet.

## Phase 0 — Prove the core idea

Your final proof should look like this:

```text
                 YOUR LAPTOP
                     │
                     ▼
              python agent.py
                     │
          URL + username + password
                     │
                     ▼
                Playwright
                     │
                     ▼
             Login to website
                     │
                     ▼
          AI explores application
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
      DOM        Screenshot      Network
       │             │             │
       └─────────────┼─────────────┘
                     ▼
                Claude Code
                     │
                     ▼
            Application Map
                     │
                     ▼
          Scenario Generation
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
   Functional   Performance     Security
                     │
                     ▼
              scenarios.json
```

---

# Step 1 — Create a completely separate project

Don't put it inside your existing QA platform yet.

Create:

```powershell
mkdir qa-agent-poc
cd qa-agent-poc
```

Create:

```text
qa-agent-poc/
│
├── agent.py
├── explorer.py
├── analyzer.py
├── requirements.txt
│
└── output/
```

This is your **experimental laboratory**.

---

# Step 2 — Verify your environment

You already have Claude Code, so first verify:

```powershell
claude --version
```

Then:

```powershell
python --version
```

You want Python 3.10+ preferably.

Then:

```powershell
node --version
```

And:

```powershell
npx playwright --version
```

If Playwright isn't installed:

```powershell
pip install playwright
playwright install chromium
```

---

# Step 3 — First experiment: browser only

Don't use Claude yet.

Create:

```text
test_browser.py
```

Put:

```python
import asyncio
from playwright.async_api import async_playwright


async def main():

    async with async_playwright() as p:

        browser = await p.chromium.launch(
            headless=False
        )

        page = await browser.new_page()

        await page.goto(
            "https://app.theaccountant.finance/login"
        )

        print("TITLE:")
        print(await page.title())

        print("URL:")
        print(page.url)

        await page.screenshot(
            path="output/test.png",
            full_page=True
        )

        await browser.close()


asyncio.run(main())
```

Run:

```powershell
python test_browser.py
```

### Expected result

Chromium opens.

You should see the application.

If this doesn't work, **stop here and fix Playwright**.

Don't move to AI yet.

---

# Step 4 — Add login

Now test:

```text
URL
 ↓
Username
 ↓
Password
 ↓
Login
 ↓
Dashboard
```

Don't make this intelligent yet.

Use Playwright selectors.

Your test should successfully reach the authenticated application.

Verify:

```python
print(page.url)
print(await page.title())
```

And take:

```text
output/
   login.png
   dashboard.png
```

---

# Step 5 — Collect application evidence

Now create:

```text
explorer.py
```

Initially collect only:

### Page

```text
URL
Title
Visible text
```

### UI

```text
Buttons
Links
Inputs
Forms
Selects
```

### Browser

```text
Screenshot
Console errors
Network requests
```

The output should be something like:

```json
{
  "url": "https://app.theaccountant.finance/dashboard",
  "title": "The Accountant",
  "buttons": [
    "Customers",
    "Transactions",
    "Reports"
  ],
  "links": [
    "/dashboard",
    "/customers",
    "/transactions"
  ],
  "inputs": [],
  "forms": 0
}
```

Save it as:

```text
output/application.json
```

---

# Step 6 — Now introduce Claude

This is the first important AI experiment.

Give Claude:

```text
application.json
```

and ask:

> Based only on this application evidence, identify potential Functional, Performance and Security test scenarios. Do not invent functionality that is not present.

Output:

```text
output/scenarios.json
```

For example:

```json
{
  "functional": [
    "Login with valid credentials",
    "Create customer",
    "Search customer",
    "Create transaction"
  ],
  "performance": [
    "Dashboard load performance",
    "Transaction processing response time"
  ],
  "security": [
    "Authentication protection",
    "Unauthorized access protection"
  ]
}
```

### This is your first milestone.

If this works, you have proved:

> **Real application → evidence → AI → test scenarios**

---

# Step 7 — Don't ask Claude to explore yet

This is where I would be careful.

Don't immediately give Claude:

> "Go explore everything."

First build a deterministic crawler.

Something like:

```text
Start page
   ↓
Find links/buttons
   ↓
Open safe candidate
   ↓
Collect evidence
   ↓
Return
   ↓
Next candidate
```

Keep a visited list:

```python
visited = set()
```

Example:

```text
/dashboard
/customers
/transactions
/reports
/settings
```

Avoid infinite loops.

---

# Step 8 — Build the Application Map

Now your agent should produce:

```json
{
  "application": "The Accountant",

  "pages": [
    {
      "url": "/dashboard",
      "title": "Dashboard"
    },
    {
      "url": "/customers",
      "title": "Customers"
    },
    {
      "url": "/transactions",
      "title": "Transactions"
    }
  ],

  "entities": [
    "Customer",
    "Transaction",
    "Journal"
  ],

  "actions": [
    "Create",
    "Edit",
    "Delete",
    "Search"
  ]
}
```

This is **much more valuable** than just screenshots.

---

# Step 9 — Add Claude's reasoning

Now Claude receives:

```text
Application Map
+
Page evidence
+
Screenshots
+
Network information
```

and determines:

```text
What is important?
What are the business workflows?
What should be tested?
What could be security-sensitive?
What could be performance-sensitive?
```

Then:

```text
Functional
Performance
Security
```

---

# Step 10 — Test scenario quality check

This is extremely important.

Don't immediately connect it to your UI.

Manually inspect:

```text
scenarios.json
```

Ask yourself:

### Is it realistic?

```text
✓ Create customer
✓ Search customer
✓ Create transaction
```

### Did AI hallucinate?

```text
❌ Cryptocurrency payment
❌ Export to SAP
❌ Mobile biometric login
```

If the application doesn't have those features, the AI shouldn't generate them.

This is one of the most important quality gates for your product.

---

# Step 11 — Add scenario confidence

I would add:

```json
{
  "name": "Create customer",
  "category": "Functional",
  "risk": "Medium",
  "confidence": 0.96,
  "evidence": [
    "/customers",
    "Add Customer button",
    "Customer form"
  ]
}
```

Now your AI isn't simply saying:

> "I think this should be tested."

It says:

> "I found these actual application elements supporting this scenario."

That's much stronger.

---

# Step 12 — Add visual AI

Only after the previous steps work.

For a page:

```text
DOM
+
Screenshot
```

give both to the AI.

Ask:

> Identify visually important interactive elements that may not be obvious from the DOM.

This can help with:

* canvas-based UI
* unusual buttons
* icons
* visual dialogs
* dynamically rendered components

But **don't make computer vision your primary discovery mechanism**.

Use:

**DOM first → accessibility → network → vision as additional evidence.**

---

# Step 13 — Add AI self-healing

Only after discovery works.

Create this experiment:

```text
Test:
Click "Add Customer"

        ↓

Change application:
"Add Customer"
→
"Create Customer"

        ↓

Original locator fails

        ↓

AI sees:
DOM + screenshot

        ↓

Finds:
"Create Customer"

        ↓

Retry

        ↓

PASS
```

If you can demonstrate this reliably, you have a serious feature.

---

# Step 14 — Add actual execution

Now take:

```text
scenarios.json
```

and convert one scenario into executable steps.

Example:

```json
{
  "name": "Create customer",
  "steps": [
    {
      "action": "navigate",
      "target": "/customers"
    },
    {
      "action": "click",
      "target": "Add Customer"
    },
    {
      "action": "fill",
      "target": "Name",
      "value": "Jordan Example"
    },
    {
      "action": "click",
      "target": "Save"
    },
    {
      "action": "verify",
      "target": "Jordan Example"
    }
  ]
}
```

Then Playwright executes it.

---

# Step 15 — Add failure analysis

When Playwright fails:

```text
Playwright error
+
Screenshot
+
DOM
+
Network
+
Previous action
```

Claude analyzes:

```text
USER DATA ERROR
APPLICATION ERROR
ENVIRONMENT ERROR
AUTOMATION ERROR
AUTHENTICATION ERROR
UNKNOWN
```

This becomes your **Errors** feature later.

---

# Step 16 — Only after all that, build the cloud platform

Once this local process works:

```text
URL
 ↓
Login
 ↓
Explore
 ↓
Application Map
 ↓
AI Scenario Generation
 ↓
Functional / Performance / Security
 ↓
Execute
 ↓
Failure Analysis
 ↓
Self Healing
```

then build:

```text
             CLOUD
        QA Finance Backend
              │
       ┌──────┴──────┐
       │             │
       ▼             ▼
    Database       Frontend
       │
       │
       ▼
    Job Queue
       │
       ▼
 LOCAL QA AGENT
       │
       ▼
    Playwright
       │
       ▼
 Customer Application
```

---

# Your development roadmap

I would literally follow this order:

```text
PHASE 0
────────────────────────
✓ Claude Code works
✓ Python works
✓ Playwright works


PHASE 1
────────────────────────
✓ Open website
✓ Login
✓ Capture screenshot
✓ Capture DOM
✓ Capture links/buttons/forms


PHASE 2
────────────────────────
✓ Explore multiple pages
✓ Build application map


PHASE 3
────────────────────────
✓ Send application map to Claude
✓ Generate Functional scenarios
✓ Generate Performance scenarios
✓ Generate Security scenarios
✓ Save JSON


PHASE 4
────────────────────────
✓ Confidence score
✓ Evidence for each scenario
✓ Remove hallucinated scenarios


PHASE 5
────────────────────────
✓ Generate executable steps
✓ Playwright execution
✓ Screenshots
✓ Video
✓ Trace


PHASE 6
────────────────────────
✓ Failure classification
✓ AI root-cause analysis
✓ Self-healing


PHASE 7
────────────────────────
✓ Local QA Agent
        ↕
✓ Cloud QA Backend


PHASE 8
────────────────────────
✓ React UI
✓ Dashboard
✓ Automation
✓ Test Cases
✓ Test Lab
✓ Errors
✓ Reports
```

## The first milestone I would target

Don't think about the whole product yet.

Get this exact command working:

```powershell
python agent.py
```

Then:

```text
Enter URL
Enter username
Enter password

       ↓

Browser opens

       ↓

Agent logs in

       ↓

Agent explores application

       ↓

output/application-map.json

       ↓

Claude analyzes

       ↓

output/scenarios.json

       ↓

┌──────────────────────────┐
│ FUNCTIONAL     32        │
│ PERFORMANCE     8        │
│ SECURITY        8        │
│                          │
│ TOTAL          48        │
└──────────────────────────┘
```


agent.py
   ↓
explorer.py
   ↓
Playwright + Chromium
   ↓
Website exploration
   ↓
exploration.json
   ↓
analyzer.py
   ↓
QA scenarios
   ↓
scenarios.json
report.json

**If you can make that reliable on 2–3 different real applications, then you have validated the core technology before spending time building the full QA platform.**
