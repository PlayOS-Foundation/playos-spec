# Store

The **Store** is where players discover, browse, and purchase
applications. It is accessible from the home screen.

## Layout

```
┌──────────────────────────────────────────┐
│  Store            🔍 Search    Cart (0)  │
├──────────────────────────────────────────┤
│                                          │
│  [Featured]  [New]  [Top]  [Categories]  │  Tabs
│                                          │
│  ┌──────────────────────────────────────┐│
│  │  ★ Featured                          ││
│  │  [Large hero with trailer/art]       ││
│  │  "Game of the Week"                  ││
│  └──────────────────────────────────────┘│
│                                          │
│  New Releases                            │
│  [App 1   ]  [App 2   ]  [App 3   ]     │
│  [$14.99  ]  [Free    ]  [$9.99   ]     │
│                                          │
│  Top Sellers                             │
│  [App 4   ]  [App 5   ]  [App 6   ]     │
│  [$4.99   ]  [$19.99  ]  [Free    ]     │
│                                          │
└──────────────────────────────────────────┘
```

## Product page

When an application is selected:

```
┌──────────────────────────────────────────┐
│  ← Store                                 │
├──────────────────────────────────────────┤
│  [Screenshots / Trailer]                 │
│                                          │
│  Game Title                              │
│  Developer Name                          │
│  ★★★★☆ (4.2)  12,500 ratings            │
│  Category: Action   Size: 2.3 GB         │
│                                          │
│  Description:                            │
│  An epic action game...                  │
│                                          │
│  Permissions:                            │
│  • Network access                        │
│  • Cloud saves                           │
│                                          │
│  [A: Purchase $14.99]  [X: Wishlist]    │
└──────────────────────────────────────────┘
```

## Purchase flow

1. Player selects "Purchase" on a product page.
2. Confirmation dialog with price, tax, and payment method.
3. Player confirms.
4. Payment is processed.
5. Download begins automatically.
6. Application appears in Library and on the Home screen.

## Search

Search is accessed from the Store header:

- Type with on-screen keyboard or physical keyboard.
- Results update as you type.
- Filter by category, price range, rating.
- Sort by relevance, new, popular, price.

## Categories

Applications are organized into categories:

| Category | Examples |
|---|---|
| Action | Platformers, shooters, fighting |
| Adventure | Point-and-click, visual novels |
| RPG | Turn-based, action RPG, roguelikes |
| Strategy | Tactics, 4X, tower defense |
| Simulation | Life sim, management, vehicle sim |
| Puzzle | Match-3, physics puzzles |
| Sports & Racing | Racing, sports sims, fitness |
| Arcade | Score-chasing, retro |
| Utilities | File managers, system tools |
| Demos | Free trials |

## Wishlist

Players can add applications to a wishlist. The wishlist:

- Sends notifications when wishlisted apps go on sale.
- Is visible to friends (optional).
- Can be shared (link or QR code).

## Related

- [Marketplace](../03-platform-architecture/09-marketplace.md)
- [Marketplace Catalog](../11-cloud-and-marketplace/07-marketplace-catalog.md)
- [Entitlements](../11-cloud-and-marketplace/09-entitlements.md)
- [Payments and Payouts](../11-cloud-and-marketplace/10-payments-and-payouts.md)
