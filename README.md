# lestat-inc-agents

LesTaT Inc. AI bot ekibi public showcase. Auto-deployed to **agents.skilldrunk.com** via Vercel. State refresh every 30 min via GitHub Actions.

## Files
- \`index.html\` — showcase page (5 bot kartı, modern dark, Cormorant Garamond)
- \`state.json\` — canlı stat'lar (sync-state workflow ile güncel)
- \`vercel.json\` — caching + CORS headers
- \`.github/workflows/sync-state.yml\` — VPS state çek + sanitize + commit (30 dk)
