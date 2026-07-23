# Analytics

The **Analytics** service collects anonymous usage data to help
developers understand how players interact with their applications.

## Data collected

| Metric | Description |
|---|---|
| Sessions | Number of sessions, session duration |
| Active users | Daily, weekly, monthly active users |
| Retention | Day 1, 7, 30 retention rates |
| Device type | Runtime Device vs SDK Target distribution |
| OS | Operating system breakdown |
| Screen resolution | Display resolution distribution |
| Language | Player language settings |
| Play time | Total and average play time |
| Achievement unlocks | Completion rate per achievement |
| Leaderboard entries | Submission volume |

## What is NOT collected

- Personally identifiable information (PII).
- Precise location.
- Raw input data.
- Screen captures or recordings.
- Application state or save data.

## Developer dashboard

Analytics are displayed in the Developer Portal:

```
┌──────────────────────────────────────────┐
│  Daily Active Users:  1,247  ↑ 12%      │
│  New Sessions Today:    842  →  +2%     │
│  Avg. Session:        42 min  ↓  -5%    │
│  Day-7 Retention:      38%   ↑  +3%     │
└──────────────────────────────────────────┘
```

## Player opt-out

Players can disable analytics in Settings → Privacy → Usage Data.
When disabled, the Runtime sends a "do not track" signal and no
analytics are collected.

## Self-hosting

The Analytics service is Prometheus-compatible. Self-hosted deployments
use Prometheus + Grafana for dashboards. The data pipeline is:
Application → Runtime → Analytics Service → Prometheus.

## Related

- [Developer Portal](05-developer-portal.md)
- [Crash Reporting](15-crash-reporting.md)
- [Observability](../08-runtime-architecture/18-observability.md)
- [Self-Hosting Principles](02-self-hosting-principles.md)
