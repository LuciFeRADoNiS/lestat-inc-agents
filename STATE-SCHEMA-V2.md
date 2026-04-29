# state.json — Schema v2

> Cowork ↔ Skilldrunk koordinasyon kontratı.
> v1 mevcut hâlâ çalışır (HTML graceful fallback yapar). v2 yeni alanları VPS maintenance script'inin doldurması beklenir.

**Tarih:** 2026-04-29
**Yazan:** Skilldrunk session (agents-deploy refresh)
**Tüketen:** index.html (yeni infografikler), Cowork VPS maintenance script v3 (üretici)

## v1 → v2 farkı (özet)

Yeni alanların hiçbiri zorunlu değil. Eksik olduğunda HTML "veri bekleniyor" placeholder gösterir, sayfa düzgün çalışmaya devam eder.

| Alan | Tip | Üretim önerisi |
|---|---|---|
| `bots[].model` | string | Maintenance config'inden çek (`claude-sonnet-4`, `gemini-2.5-flash`, `suno-api`, `claude-haiku-4-5`) |
| `bots[].ram_history` | number[7] | Her gün cron snapshot al (`ps -o rss`), son 7 günü kaydır |
| `bots[].cost_7d_usd` | number | Anthropic billing API + Gemini billing CSV'sinden bot codename ile filtrele, son 7g |
| `ai_cost_history` | `{date, usd}[30]` | Aynı kaynaklardan günlük toplam, son 30g |
| `crashes` | `{ts, codename, kind}[]` | systemd journal grep + restart marker, son 7g (kind: `"crash"` \| `"restart"`) |
| `automations_recent` | `{ts, type, summary}[]` | Bot log feed'leri (cron run, NLP intent, dispatch, brief produced) son 24h, max 50 entry |

## Tam v2 örneği

```json
{
  "schema_version": "2.0",
  "lestat_inc_version": "2.0",
  "snapshot_at": "2026-04-29T17:30:00Z",
  "vps": {
    "uptime": "70 days",
    "disk_used": "16%",
    "load_1m": "0.12"
  },
  "stats": {
    "active_bots": 5,
    "total_ram_mb": 916,
    "today_api_cost_usd": 0.74,
    "automation_count": 27,
    "openclaw_version": "2026.4.25",
    "openclaw_latest": "2026.4.25",
    "crashes_last_7d": 4
  },
  "bots": [
    {
      "codename": "Hermes",
      "role": "Chief Orchestrator",
      "model": "Claude Sonnet 4",
      "status": "live",
      "phase": 4,
      "ram_mb": 38,
      "ram_history": [42, 39, 41, 38, 40, 38, 38],
      "cost_7d_usd": 1.42
    },
    {
      "codename": "Atlas",
      "role": "Strategic Brain",
      "model": "Claude Sonnet 4",
      "status": "live",
      "ram_mb": 125,
      "ram_history": [120, 118, 124, 130, 128, 125, 125],
      "cost_7d_usd": 2.10
    },
    {
      "codename": "Mnemosyne",
      "role": "Personal Companion",
      "model": "Gemini 2.5 Flash",
      "status": "live",
      "ram_mb": 660,
      "ram_history": [640, 655, 670, 665, 660, 658, 660],
      "cost_7d_usd": 0.84
    },
    {
      "codename": "Hephaestus",
      "role": "Operations Floor",
      "model": "Claude Sonnet 4",
      "status": "live",
      "ram_mb": 68,
      "ram_history": [62, 65, 70, 68, 66, 68, 68],
      "cost_7d_usd": 12.40
    },
    {
      "codename": "Apollo",
      "role": "Music Producer",
      "model": "Suno API",
      "status": "live (sunset planned)",
      "ram_mb": 0,
      "ram_history": [0,0,0,0,0,0,0],
      "cost_7d_usd": 0.15
    },
    {
      "codename": "Calliope",
      "role": "Skilldrunk Brand Voice",
      "model": "Claude Haiku 4.5",
      "status": "live",
      "ram_mb": 0,
      "ram_history": [0,0,0,0,0,0,0],
      "cost_7d_usd": 0.42
    }
  ],
  "ai_cost_history": [
    { "date": "2026-03-31", "usd": 1.12 },
    { "date": "2026-04-01", "usd": 1.34 },
    { "date": "2026-04-02", "usd": 0.98 }
  ],
  "crashes": [
    { "ts": "2026-04-26T14:22:00Z", "codename": "Mnemosyne", "kind": "crash" },
    { "ts": "2026-04-26T14:22:30Z", "codename": "Mnemosyne", "kind": "restart" },
    { "ts": "2026-04-28T03:14:00Z", "codename": "Atlas",     "kind": "restart" }
  ],
  "automations_recent": [
    { "ts": "2026-04-29T08:00:00Z", "type": "cron",     "summary": "Atlas daily brief produced" },
    { "ts": "2026-04-29T09:15:00Z", "type": "intent",   "summary": "Hermes dispatched 'add Toyota call to calendar' to MasteRMinD" },
    { "ts": "2026-04-29T12:00:00Z", "type": "summary",  "summary": "Hephaestus group digest posted" },
    { "ts": "2026-04-29T13:42:00Z", "type": "skilldrunk","summary": "Calliope answered /ask 'son 7g pageview' via query_db" }
  ]
}
```

## Calliope için Skilldrunk-side feed (opsiyonel)

Calliope (skilldrunk Telegram botu) VPS'te değil — Vercel edge'inde. Bu yüzden VPS maintenance script onun verisini doğrudan üretemez. İki seçenek:

**A) Cowork tarafında stub:** Maintenance script `bots[]`'a Calliope için sabit row ekler (`ram_mb: 0`, `cost_7d_usd: null`, `status: "live"`). HTML zaten Calliope yoksa fallback ekliyor; bu seçenekte Cowork sadece bilgilendirme amaçlı satır ekler.

**B) Skilldrunk-side cron (önerilen, ileri tarihli):**
`apps/admin/src/app/api/cron/calliope-state/route.ts`:
- Daily 04:00 UTC çalışır
- `sd_ai_usage` tablosundan son 7g `app='admin-ai'` veya bot tag'li çağrılar toplanır
- HTTP POST `https://agents.skilldrunk.com/api/state-merge` (Cowork tarafına yeni endpoint) → Calliope dilimi enjekte
- Veya GitHub API ile `lestat-inc-agents` repo'suna `calliope-state.json` commit, Cowork merge

B varyantı bir sonraki sprint için. v2 ship'lerken A yeterli.

## Backward compat

`schema_version: "1.0"` döndüğünde v2 HTML şu davranışı gösterir:
- Hero stats: aynı (mevcut alanlar)
- Donut/Model bars: bots[] varsa render, model field yoksa fallback'ten enrich
- Sparklines: ram_history yoksa "v3 schema ile gelir" placeholder
- Cost 7g: cost_7d_usd yoksa placeholder
- Cost trend: ai_cost_history yoksa today_api_cost_usd × 30 ile düz sentetik çizgi (opacity düşürülmüş + "tahmin" etiketi)
- Crash timeline: crashes yoksa placeholder
- Automation feed: automations_recent yoksa placeholder

Yani üretici v3 ship etmeden de v2 HTML çalışır, sadece "yarım dolu" kalır.

## Üretici (maintenance script v3) için ipuçları

1. **`ram_history`**: VPS'te zaten her dakika `top` log'u var. Daily cron'da son 24h ortalamasını al, son 7 günlük değeri kaydır (FIFO). Her bot için `proc_name` filter ile.
2. **`cost_7d_usd`**: Anthropic API'de bot başına ayrı API key kullanmıyorsanız (kullanıyorsanız organization summary endpoint), bot codename'i her request'te `metadata` field'ında gönder, sonra Anthropic billing export'tan toplama yap. Gemini için Cloud Console billing CSV.
3. **`ai_cost_history`**: Yukarıdaki toplama günlük yapılıp 30g pencere kaydırılır.
4. **`crashes`**: `journalctl -u <service> --since "7 days ago" | grep -E "(restart|crashed|OOM)"` parse et, ts + codename + kind ile array'e dönüştür.
5. **`automations_recent`**: Her bot kendi log'una ts + tip + 1-cümle özet yazsın (cron başlangıcı, intent dispatch, brief produced). Maintenance script bu log'ları aggregate eder, son 24h'i alır.

## Versiyonlama

- HTML her zaman v1 + v2 hybrid okur — opaklık-göstergesi `state.lestat_inc_version` üzerinden.
- v3'e geçince `schema_version: "3.0"` olur, breaking olmadan ek alan eklenebilir.
- Breaking değişiklik gerekirse: yeni endpoint (`/state.v3.json`), HTML eski URL'i fallback olarak tutar.
