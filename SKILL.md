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

**If Slack returns no results:** Retry once before showing an error.

## Step 2: Show SAE Selector Artifact

**IMMEDIATELY create a React artifact** displaying a grid of SAE cards (name, title, email). Clicking a card triggers account analysis.

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
   - Rank number with score-based color gradient
   - Company name
   - Badge: "FOCUS" or "TARGET"
   - Score (0-10)
   - Expandable details:
     - Score breakdown by criteria
     - Current CI/CD tools
     - Key triggers
     - Target personas
     - Suggested outreach angle
     - Source links

3. Buttons: "Re-analyze" and "Change SAE"

## CRITICAL UI RULES — MUST FOLLOW

**Use ONLY inline styles. Do NOT use Tailwind classes.**

CircleCI Brand Colors:
- Page background: `background: '#1C273A'`
- Card background: `background: '#2E3C52'`
- Primary accent (titles, highlights): `color: '#00DB74'`
- Headlines: `color: '#FFFFFF'`
- Body text: `color: '#FAFAFA'`
- Secondary text: `color: '#D4D4D4'`
- FOCUS badge: `background: '#FC5656', color: '#FFFFFF'`
- TARGET badge: `background: '#0057FF', color: '#FFFFFF'`

Typography:
- Headings: `fontFamily: 'Space Grotesk, sans-serif'`
- Body: `fontFamily: 'Inter, system-ui, sans-serif'`

Example inline style for a card:
```jsx
<div style={{ 
  background: '#2E3C52', 
  borderRadius: '12px', 
  padding: '16px',
  color: '#FAFAFA'
}}>
```

**NEVER use:**
- Tailwind classes (bg-slate-900, text-white, etc.)
- CSS files or <style> tags
- Default browser colors

## Error Handling

| Error | Response |
|-------|----------|
| Slack unavailable | "Please enable Slack connector in Claude Settings" |
| Slack returns empty | Retry once, then show error if still empty |
| HubSpot unavailable | "Please enable HubSpot connector in Claude Settings" |
| No accounts found | "No Focus or Target accounts found for this SAE" |

## FINAL CHECKLIST

1. ✅ Always output a React artifact — never just text
2. ✅ Use ONLY inline styles with CircleCI brand colors
3. ✅ Dark background (#1C273A), light text (#FAFAFA)
4. ✅ Green accent (#00DB74) for titles and highlights
5. ✅ Red FOCUS badge (#FC5656), Blue TARGET badge (#0057FF)
6. ✅ Show progress during analysis
7. ✅ Make account cards expandable
