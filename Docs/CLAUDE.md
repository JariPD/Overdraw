# Overdraw — Claude Code Context

This file gives Claude Code full awareness of the project. Read it before making any changes.

---

## What this project is

Overdraw is a real-time multiplayer Blackjack web app. Up to 8 players join a shared table via a room code and play against the dealer simultaneously. All game state is broadcast live via SignalR. Built as a portfolio project by Jari Dijk (jaridijk.nl).

---

## Tech stack

| Layer | Technology |
|---|---|
| Framework | ASP.NET Core 8, Razor Pages |
| Real-time | SignalR (core feature — treat with care) |
| ORM | Entity Framework Core, code-first |
| Database | SQLite (`overdraw.db`) |
| Styling | Tailwind CSS 4 — CSS-first, `@tailwindcss/cli` build pipeline (no `tailwind.config.js`, sources auto-detected) |
| JS | Vanilla JS + SignalR JS client |
| Testing | xUnit, Moq, FluentAssertions |

No frontend framework (React, Vue, etc.). No authentication. No real money.

---

## Project structure

```
Overdraw/
├── Overdraw.sln
├── src/
│   └── Overdraw.Web/
│       ├── Hubs/GameHub.cs           ← SignalR hub, thin router only
│       ├── Services/
│       │   ├── Interfaces/           ← IGameService, IDeckService, IRoomManager
│       │   ├── GameService.cs        ← all game logic lives here
│       │   ├── DeckService.cs        ← shuffle and deal only
│       │   └── RoomManager.cs        ← ConcurrentDictionary, Singleton
│       ├── Models/                   ← Card, Hand, Player, Game, Enums
│       ├── Data/                     ← AppDbContext, GameRecord entity
│       ├── Pages/                    ← Index.cshtml, Game.cshtml
│       └── wwwroot/
│           ├── css/input.css         ← Tailwind source
│           └── js/game.js            ← SignalR JS client
└── tests/
    └── Overdraw.Tests/
```

---

## Architecture rules — always follow these

**GameHub is a thin router.** It receives SignalR calls, delegates to `IGameService` and `IRoomManager`, then broadcasts results. Zero game logic in the hub.

**Services depend on interfaces, never concretions.** `GameHub` injects `IGameService` and `IRoomManager`. `GameService` injects `IDeckService`. Never `new` a service manually.

**DI lifetimes:**
- `IRoomManager` → `Singleton` (shared in-memory state)
- `IGameService` → `Scoped`
- `IDeckService` → `Scoped`

**Game state is in-memory during play.** `RoomManager` holds a `ConcurrentDictionary<string, Game>`. Only completed rounds are persisted to SQLite via `GameRecord`.

**The name "Overdraw" must not appear in game logic, services, or models.** Only in: solution name, `.csproj` namespace, `_Layout.cshtml` title, `README.md`. This makes renaming a 2-minute job.

---

## SignalR event contract

### Client → Server (Hub methods)
| Method | Payload |
|---|---|
| `JoinRoom` | roomCode, playerName |
| `StartGame` | roomCode |
| `Hit` | roomCode |
| `Stand` | roomCode |

### Server → Client (broadcasts)
| Event | Payload |
|---|---|
| `PlayerJoined` | playerName, playerCount |
| `GameStarted` | players[], dealerCard, currentPlayer |
| `CardDealt` | playerName, card, playerStatus |
| `TurnChanged` | playerName |
| `DealerCardRevealed` | cards[] |
| `DealerHit` | card, total |
| `RoundOver` | results[] |
| `PlayerLeft` | playerName |

Do not add new events without updating this file.

---

## Game rules (source of truth)

- 52-card deck, reshuffled each round
- Number cards = face value, face cards = 10, Ace = 11 or 1 (auto-adjusted)
- Dealer hits until 17 or higher, then stands
- Blackjack = Ace + 10-value card on initial deal
- All players play against the dealer, not each other
- Turn order: left to right, dealer last
- Results: Blackjack > Win > Push > Lose > Bust

---

## Coding conventions

- C# 12, nullable enabled, file-scoped namespaces
- Records for immutable value types (`Card`)
- `string.Empty` not `""`
- No logic in constructors — use factory methods or services
- Every public method on a service has a corresponding unit test
- Errors in hub methods throw `HubException`, not generic exceptions

---

## What's not built yet (backlog lives in GitHub Issues)

- Frontend UI (Razor Pages + Tailwind + game.js)
- EF Core migration
- Deployment config

---

## Running locally

```bash
# Terminal 1 — CSS watch
cd src/Overdraw.Web
npm run css:watch

# Terminal 2 — App
dotnet run --project src/Overdraw.Web

# Tests
dotnet test
```

---

## Important: do not

- Put game logic in `GameHub`
- Use `new GameService(...)` outside of DI registration
- Add jQuery or Bootstrap (removed intentionally)
- Hardcode the room code format anywhere except `RoomManager`
- Commit `wwwroot/css/app.css` (it's a build artifact)

