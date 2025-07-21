#this dirctory is for the hl-grid bot
                           ┌──────────────────────────────┐
                           │        Telegram Client        │
                           └────────────┬─────────────────┘
                                        │
                          ┌─────────────▼──────────────┐
                          │       Telegram Bot API      │
                          └─────────────┬──────────────┘
                                        │
                    ┌──────────────────▼──────────────────┐
                    │         Bot Command Router           │
                    │  (e.g., /start, /grid, /xp, /refer)  │
                    └──────────────────┬──────────────────┘
                                       │
       ┌───────────────────────────────▼─────────────────────────────┐
       │                      Business Logic Layer                    │
       │  - Grid Trading Engine       - XP + Volume Tracker           │
       │  - Referral System           - User Session Manager          │
       └──────────────┬────────────────────┬─────────────────────────┘
                      │                    │
         ┌────────────▼──────────┐  ┌──────▼──────────┐
         │ Hyperliquid Python SDK│  │  Database (Postgres / Redis) │
         └────────────┬──────────┘  └──────┬──────────┘
                      │                    │
              ┌───────▼───────┐    ┌───────▼────────┐
              │ Order Placement│    │ User XP / Grid│
              │ Cancel / Modify│    │ Referrals / LP│
              └───────────────┘    └────────────────┘
