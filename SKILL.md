---
name: sae-account-prioritizer
description: Prioritizes CircleCI SAE accounts for weekly outreach. Triggers on "prioritize accounts", "prioritize my accounts", "weekly priorities", "territory review", or "which accounts should I focus on".
---

# SAE Account Prioritizer

**IMPORTANT: This skill MUST output a React artifact with the full visual interface.**

## Step 1: Get SAE Roster from Slack

```
Slack:slack_search_users
  query: "Strategic Account Executive"
  limit: 20
```

Filter to ONLY these exact titles:
- "Strategic Account Executive"
- "Strategic Account Executive, NAMER"
- "Strategic Account Executive, EMEA"  
- "Strategic Account Executive, JAPAC"

EXCLUDE: "Client", "Growth", "Senior" variants.

Save: name, email, slackId, title

**If Slack returns no results or empty response:** Retry the search once with the same query before showing an error. Slack API can occasionally return empty on first attempt.

## Step 2: Show SAE Selector Artifact

**IMMEDIATELY create a React artifact** that displays:
- Grid of SAE cards (name, title, email)
- Clicking a card triggers account analysis

Use CircleCI brand colors:
- Primary green: #00DB74
- Dark background: #1C273A
- Light background: #EDEDED

## Step 3: When User Selects an SAE

Get their HubSpot owner ID:
```
HubSpot:search_owners
  searchQuery: "{sae_email}"
```

## Step 4: Fetch Accounts from HubSpot

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

Focus = big_bet is true
Target = hs_is_target_account is true

## Step 5: Score Accounts

Use web_search to research each company. Score 1-10 on:

| Criteria | Weight |
|----------|--------|
| CI/CD Need | 30% |
| Current Solution | 25% |
| Market Timing | 20% |
| Company News | 15% |
| Reachable Contacts | 10% |

Focus accounts get +0.5 bonus.

## Step 6: Display Results in Artifact

**UPDATE the React artifact** to show:

1. Summary bar: Total / Focus / Target counts
2. Ranked list of Top 10 accounts with:
   - Rank number with score-based color (green→blue gradient)
   - Company name
   - Badge: "FOCUS" (red #FC5656) or "TARGET" (blue #0057FF)
   - Score (0-10)
   - Expandable details:
     - Score breakdown by criteria
     - Current CI/CD tools
     - Key triggers
     - Target personas
     - Suggested outreach angle
     - Source links

3. Buttons: "Re-analyze" and "Change SAE"

## Error Handling

| Error | Response |
|-------|----------|
| Slack unavailable | "Please enable Slack connector in Claude Settings" |
| Slack returns empty | Retry once, then show error if still empty |
| HubSpot unavailable | "Please enable HubSpot connector in Claude Settings" |
| No accounts found | "No Focus or Target accounts found for this SAE" |

## CRITICAL RULES

1. **ALWAYS output a React artifact** - never just text
2. Start with SAE selector, update with results
3. Use CircleCI brand colors (#00DB74, #1C273A, #EDEDED)
4. Show progress during analysis
5. Make account cards expandable
6. Retry Slack search once if it returns empty
