# Loyalty — share folder

| file | what |
|---|---|
| `loyalty_status_ui_mockup_{members,expiring,schemes}.png` | Loyalty Status report, three tabs (menu 79) |
| `loyalty_status_ui_mockup.py` | renders the three PNGs |
| `plan-backend-loyalty-status.md` | backend plan for Prathap: `/api/v1/reports/loyalty-status/*`, findings F1–F4, decisions D1–D7 |

Rules: all logic in NestJS (no DB functions/triggers/views). SMS module: decided later.
