# Overdraw — Design Document
**Project:** Overdraw  
**Stack:** ASP.NET Core 8 · SignalR · Entity Framework Core · SQLite · Tailwind CSS 4  
**Repo:** github.com/jaridijk/overdraw  
**Portfolio:** jaridijk.nl  
**Status:** In development

---

## 1. Overview

Overdraw is a real-time multiplayer Blackjack web app. Up to 8 players join a shared table and play against the dealer simultaneously. All players see each other's cards, actions, and results live via SignalR. Built as a portfolio piece demonstrating full-stack .NET, real-time communication, and clean UI.

---

## 2. Goals

- Learn and demonstrate SignalR for real-time bidirectional communication
- Build a clean ASP.NET Core backend with SOLID principles and proper separation of concerns
- Practice Entity Framework Core with SQLite for game history persistence
- Style a polished UI with Tailwind CSS
- Deploy free and feature on jaridijk.nl

---

## 3. Features

### MVP
- Create or join a room with a 4-letter room code
- Up to 8 players per table
- Standard Blackjack rules (hit, stand, bust, blackjack, dealer logic)
- Real-time updates: all players see all cards and actions as they happen
- Turn-based flow: players act one at a time, others watch live
- Results screen per player (Win / Lose / Push / Blackjack)
- Play Again restarts a round in the same room

### Post-MVP
- Virtual chip / bet system (no real money)
- Player avatars (initials + assigned colour)
- In-room chat via SignalR
- Game history page (last 10 rounds saved to DB)
- Spectator mode

### Out of scope
- User accounts / authentication
- Real money or payments
- Mobile native app

---

## 4. Game Rules

- Standard 52-card deck, reshuffled each round
- Number cards = face value, face cards = 10, Ace = 11 or 1
- Goal: beat the dealer without exceeding 21
- Dealer hits until 17 or higher
- Blackjack (Ace + 10-value on initial deal) wins immediately
- All players play against the dealer independently — not against each other
- Turn order: players act left to right, dealer acts last

---

## 5. Project Structure

```
Overdraw/
├── Overdraw.sln
├── src/
│   └── Overdraw.Web/
│       ├── Hubs/
│       │   └── GameHub.cs              # SignalR hub — thin router only
│       ├── Services/
│       │   ├── Interfaces/
│       │   │   ├── IGameService.cs
│       │   │   ├── IDeckService.cs
│       │   │   └── IRoomManager.cs
│       │   ├── GameService.cs
│       │   ├── DeckService.cs
│       │   └── RoomManager.cs          # Singleton in-memory room registry
│       ├── Models/
│       │   ├── Game.cs
│       │   ├── Player.cs
│       │   ├── Card.cs
│       │   ├── Hand.cs
│       │   └── Enums/
│       │       ├── GameStatus.cs
│       │       └── PlayerStatus.cs
│       ├── Data/
│       │   ├── AppDbContext.cs
│       │   └── Entities/
│       │       └── GameRecord.cs
│       ├── Pages/
│       │   ├── Index.cshtml            # Landing: name + room code
│       │   └── Game.cshtml             # Table view
│       └── wwwroot/
│           ├── css/
│           │   ├── input.css           # Tailwind source
│           │   └── app.css             # Tailwind output (gitignored)
│           └── js/
│               └── game.js             # SignalR JS client
└── tests/
    └── Overdraw.Tests/
        ├── GameServiceTests.cs
        └── DeckServiceTests.cs
```

---

## 6. SignalR Contract

### Client → Server
| Method | Payload | Description |
|---|---|---|
| `JoinRoom` | roomCode, playerName | Join or create a room |
| `StartGame` | roomCode | Host starts the round |
| `Hit` | roomCode | Active player draws a card |
| `Stand` | roomCode | Active player ends their turn |

### Server → Client
| Event | Payload | Description |
|---|---|---|
| `PlayerJoined` | playerName, playerCount | New player joined lobby |
| `GameStarted` | players[], dealerCard, currentPlayer | Initial deal complete |
| `CardDealt` | playerName, card, playerStatus | Card added to a hand |
| `TurnChanged` | playerName | Next player's turn |
| `DealerCardRevealed` | cards[] | Dealer's hole card flipped |
| `DealerHit` | card, total | Dealer drew a card |
| `RoundOver` | results[] | Win/Lose/Push per player |
| `PlayerLeft` | playerName | Player disconnected |

---

## 7. Data Models

```csharp
// In-memory (not persisted during play)
public record Card(string Suit, string Value);

public class Hand
{
    public IReadOnlyList<Card> Cards { get; }
    public int Value { get; }       // Ace counted as 11 or 1
    public bool IsBust { get; }
    public bool IsBlackjack { get; }
}

public class Player
{
    public string ConnectionId { get; set; }
    public string Name { get; set; }
    public Hand Hand { get; set; }
    public PlayerStatus Status { get; set; }
}

public class Game
{
    public string RoomCode { get; set; }
    public List<Player> Players { get; set; }
    public List<Card> Deck { get; set; }
    public Hand DealerHand { get; set; }
    public GameStatus Status { get; set; }
    public int CurrentPlayerIndex { get; set; }
    public Player? CurrentPlayer { get; }
}

// Persisted after each round
public class GameRecord
{
    public int Id { get; set; }
    public string RoomCode { get; set; }
    public DateTime PlayedAt { get; set; }
    public int PlayerCount { get; set; }
    public string ResultsJson { get; set; }
}
```

---

## 8. Page Flow

```
/ (Landing)
  └─ Enter name + room code → /game/{roomCode}

/game/{roomCode} — Lobby
  └─ Players join, see each other's names
  └─ First player (host) sees "Start Game" button

/game/{roomCode} — Playing
  └─ All hands visible to all players
  └─ Active player: Hit / Stand buttons enabled
  └─ Others: buttons disabled, watch in real time
  └─ Dealer plays after all players are done

/game/{roomCode} — Results
  └─ Win / Lose / Push / Blackjack shown per player
  └─ "Play Again" → new round, same room
```

---

## 9. Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Framework | ASP.NET Core 8 | Razor Pages |
| Real-time | SignalR | Core learning goal |
| ORM | Entity Framework Core | Code-first migrations |
| Database | SQLite | Zero config, free |
| Styling | Tailwind CSS 4 | CSS-first, `@tailwindcss/cli` build pipeline |
| JS | Vanilla JS + SignalR JS client | No frontend framework |
| Testing | xUnit + Moq + FluentAssertions | Game logic unit tests |
| Hosting | Railway or Render | Free tier |
| Version control | GitHub | Public, linked from portfolio |

---

## 10. Development Phases

| Phase | Focus | Done when |
|---|---|---|
| 1 | Models + DeckService + GameService + tests | All game rules pass unit tests |
| 2 | RoomManager + GameHub + DI wiring | Hub methods broadcast correctly |
| 3 | Razor Pages + Tailwind + game.js | Full round playable in browser |
| 4 | EF Core + GameRecord persistence | Completed rounds saved to SQLite |
| 5 | Polish, error handling, deploy | Live on Railway, linked on jaridijk.nl |

---

## 11. Stretch Goals

- Virtual chip betting system
- Player avatars (initials + colour)
- In-game chat via SignalR
- Animated card dealing (CSS transitions)
- Sound effects

---

*Last updated: August 2026*
