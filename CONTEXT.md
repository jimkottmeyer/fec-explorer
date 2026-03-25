# FEC Campaign Finance Explorer — Project Context

## What this is
A single-file HTML app (`index.html`) that lets users ask plain-English questions
about U.S. campaign finance data. No build step, no dependencies, no backend.
Deployable directly to GitHub Pages.

## How it works
1. User enters their Anthropic API key (stored in localStorage, never sent anywhere else)
2. User types a natural language question (e.g. "How much did Fetterman raise in 2022?")
3. App tries the **FEC public API directly** first for common ranking queries (~1 second, free)
4. Falls back to **Claude Sonnet + web_search tool** for everything else
5. Displays a plain-English summary + formatted data table

## Tech stack
- Pure HTML/CSS/JS — single file, zero dependencies
- Anthropic API: `claude-sonnet-4-20250514` with `web_search_20250305` tool
- Requires `anthropic-dangerous-direct-browser-access: true` header for browser CORS
- FEC public API: `https://api.open.fec.gov/v1/` with `DEMO_KEY`
- All user data (query log, suggestions, API key) stored in localStorage

## Current known bug — TOP PRIORITY
Queries that fall through to Stage 2 (Claude + web search) are not returning responses.
Suspected causes:
- Claude's response may not be returning a text block (only tool_use blocks)
- JSON parsing may be failing silently
- A debug panel has been added to the page — click "Show debug log" after a query to see
  the HTTP status, stop_reason, number of content blocks, and raw response preview

The fix likely involves handling the case where `stop_reason === 'tool_use'` and making
a second API call with the tool results to get the final text response. Claude with tools
requires a multi-turn exchange: first call returns tool_use block, app submits tool results,
second call returns the final text answer.

## Features already built
- Two-stage query: FEC direct API → Claude fallback
- Animated progress bar
- Query log (localStorage, up to 200 entries, CSV export)
- Suggestion box (localStorage, with clear-all)
- Light color theme (warm off-white bg, navy text, blue accent)
- Party badges (R = red, D = blue)
- Money formatting ($12.3M, $450K)
- Responsive layout

## File structure
```
fec-explorer/
  index.html     # entire app — HTML, CSS, and JS in one file
  CONTEXT.md     # this file
  README.md      # GitHub Pages setup instructions
```

## Deployment
GitHub Pages — push to main branch, enable Pages in repo Settings.
Live at: https://[username].github.io/fec-explorer/

## Suggested next steps (in priority order)
1. Fix Stage 2 multi-turn tool use so Claude+web search queries actually return answers
2. Add specific candidate lookup (e.g. "How much did Fetterman raise?") via FEC API directly
3. Add PAC/committee search via FEC API
4. Consider adding a chart (bar chart of top fundraisers)
5. Export results to CSV
6. Share a query result as a URL

## Design notes
- Fonts: Syne (display) + DM Mono (monospace) from Google Fonts
- Color palette defined in CSS :root variables — easy to change
- All UI components use CSS variables for theming
- Mobile responsive with media queries at 600px and 700px

## API reference
The key API call pattern that needs fixing — Claude with tools requires two turns:

```javascript
// Turn 1: send question, get tool_use block back
const turn1 = await fetch('https://api.anthropic.com/v1/messages', {
  body: JSON.stringify({
    model: 'claude-sonnet-4-20250514',
    tools: [{ type: 'web_search_20250305', name: 'web_search' }],
    messages: [{ role: 'user', content: question }]
  })
});
// turn1 response has stop_reason: 'tool_use' and a tool_use content block

// Turn 2: submit tool results, get final text answer
const turn2 = await fetch('https://api.anthropic.com/v1/messages', {
  body: JSON.stringify({
    model: 'claude-sonnet-4-20250514',
    tools: [{ type: 'web_search_20250305', name: 'web_search' }],
    messages: [
      { role: 'user', content: question },
      { role: 'assistant', content: turn1.content },  // include the tool_use blocks
      { role: 'user', content: toolResults }           // submit the tool results
    ]
  })
});
// turn2 response has stop_reason: 'end_turn' and a text content block with the answer
```

Note: The web_search tool is handled server-side by Anthropic — you don't actually
execute the search yourself. Just pass the full assistant message back and Anthropic
handles the rest. Check the actual stop_reason and content block types to implement this correctly.
