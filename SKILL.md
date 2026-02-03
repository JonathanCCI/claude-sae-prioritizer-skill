---
name: sae-account-prioritizer
description: Prioritizes CircleCI Strategic Account Executive (SAE) accounts for weekly outreach using live Slack roster, HubSpot data, and market intelligence. ALWAYS outputs an interactive React artifact with CircleCI branding. Automatically rotates accounts weekly — won't show the same Top 10 two weeks in a row. Triggers on "prioritize accounts", "territory review", "weekly priorities", "which accounts to focus on", "SAE list", or when helping an SAE decide outreach targets.
---

# SAE Account Prioritizer

Analyze and prioritize Strategic Account Executive accounts for weekly outreach. **Always outputs a full interactive React artifact** with live data from Slack/HubSpot and market intelligence from web search. **Automatically rotates accounts** so you don't see the same Top 10 each week.

## Workflow

### Step 1: Get Active SAE Roster from Slack

Slack is the source of truth for active employees. HubSpot may contain departed staff.

```
Slack:slack_search_users
  query: "Strategic Account Executive"
  limit: 20
```

**CRITICAL FILTER:** Only include users whose title matches EXACTLY:
- `Strategic Account Executive`
- `Strategic Account Executive, NAMER`
- `Strategic Account Executive, EMEA`
- `Strategic Account Executive, JAPAC`

**EXCLUDE** these title variations:
- Strategic **Client** Account Executive
- Strategic **Growth** Account Executive
- **Senior** Strategic Account Executive

Extract: `name`, `email`, `slackId` (User ID field), `title`, `timezone`

### Step 2: Identify Target SAE

If user specified a name → match against Slack roster from Step 1
If not specified → check if current user's email matches an SAE

Then get their HubSpot owner ID:
```
HubSpot:search_owners
  searchQuery: "{sae_email}"
```

Required for Step 3: `ownerId`

### Step 3: Fetch Accounts from HubSpot

```
HubSpot:search_crm_objects
  objectType: "companies"
  filterGroups:
    - filters:
        - propertyName: "hubspot_owner_id"
          operator: "EQ"
          value: "{ownerId}"
        - propertyName: "big_bet"
          operator: "EQ"
          value: "true"
    - filters:
        - propertyName: "hubspot_owner_id"
          operator: "EQ"
          value: "{ownerId}"
        - propertyName: "hs_is_target_account"
          operator: "EQ"
          value: "true"
  properties: ["name", "domain", "big_bet", "hs_is_target_account", "industry"]
  limit: 200
```

Paginate with `offset` if more than 200 accounts.

**Classification:**
- **Focus**: `big_bet = true` (Big Bet program)
- **Target**: `hs_is_target_account = true` AND `big_bet != true`

### Step 4: Analyze with Web Search + AI

Search for market intelligence on accounts, then score per `references/scoring-criteria.md`.

### Step 5: Generate React Artifact

**ALWAYS** output results as an interactive React artifact using the complete template in `references/react-artifact-template.md`.

**Data injection process:**
1. Copy the full React component from `references/react-artifact-template.md`
2. Replace `/*__SAE_DATA_INJECTION_POINT__*/` with the SAE roster from Step 1:
   ```javascript
   { name: "Name", email: "email@circleci.com", title: "Strategic Account Executive, REGION", slackId: "UXXXXXX", hubspotOwnerId: "12345678" },
   ```
3. Replace `/*__ACCOUNTS_DATA_INJECTION_POINT__*/` with accounts from Step 3:
   ```javascript
   "12345678": [
     {"id":"123","name":"Company","domain":"company.com","isFocusAccount":true,"isTargetAccount":true},
   ],
   ```
4. Output the complete artifact with all data embedded

**Required UI features (must be present):**
- Chrome 143+ browser check with fallback message
- SAE selection grid with name, title, email
- Account summary cards (Total, Focus, Target counts)
- Progress indicator during analysis (3 steps)
- Ranked results with expandable cards
- Score color gradient (red→blue by rank)
- Focus badge (red) / Target badge (blue)
- Score breakdown by criteria with icons
- Current CI/CD, Key triggers, Target personas sections
- Suggested outreach angle
- Clickable source links
- Re-analyze and Change person buttons
- CircleCI brand colors and typography (Inter + Space Grotesk)
- **Weekly rotation indicator** showing new vs. continued accounts
- **Continued Priority section** for high-scoring accounts from last week

**Do NOT:**
- Output plain text summaries
- Skip the artifact
- Simplify the UI

### Step 6: Weekly Account Rotation

The artifact uses **persistent storage** to track recommendations week-over-week.

**Storage key:** `sae-priorities:{hubspotOwnerId}`

**Storage value:**
```javascript
{
  "weekOf": "2026-01-06",      // Monday of the analysis week
  "accountIds": ["id1", "id2", ...],  // This week's Top 10 IDs
  "analysisDate": "2026-01-07"
}
```

**Rotation logic (implemented in artifact):**

1. **Load previous week** from storage on analysis start
2. **Score ALL accounts** normally with web search
3. **Split results into two groups:**

   **Group A — NEW THIS WEEK (ranks #1-10):**
   Top 10 highest-scoring accounts that were NOT in last week's Top 10
   
   **Group B — CONTINUED PRIORITY (ranks #11, #12, etc.):**
   Accounts from last week that STILL score high enough to be in theoretical Top 10
   - Display with amber "CONTINUED" badge
   - Include reason: "Still high priority because: {key trigger or news}"

4. **Save to storage:** This week's Group A account IDs
5. **Display:** Group A first (#1-10), then Group B (#11+) with visual distinction

**First-time use:** If no previous data, show Top 10 normally

**User override:** If user says "include last week's accounts" or "ignore rotation", disable the filter and show pure Top 10

## Error Handling

| Error | Action |
|-------|--------|
| Slack unavailable | Ask user to enable Slack connector in Settings |
| HubSpot unavailable | Ask user to enable HubSpot connector in Settings |
| No SAEs found | Verify Slack search, check title filter |
| No accounts for SAE | Verify HubSpot owner ID, check account filters |
| Web search fails | Proceed with available data, note limitations |
| Storage fails | Continue without rotation, note in UI |

## Files

- `references/scoring-criteria.md` — 5-factor scoring rubric
- `references/react-artifact-template.md` — Chrome 143+ React component with storage
