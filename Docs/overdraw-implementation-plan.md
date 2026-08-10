# Overdraw — Implementation Plan
**Stack:** ASP.NET Core 8 · SignalR · Entity Framework Core · SQLite · Tailwind CSS 4  
**Principles:** SOLID throughout

---

## SOLID Applied

| Principle | Implementation |
|---|---|
| **S** Single Responsibility | `GameService` = game logic only. `DeckService` = cards only. `RoomManager` = room registry only. `GameHub` = thin router, zero logic. |
| **O** Open/Closed | `IDeckService` allows swapping in a multi-deck shoe without touching `GameService`. |
| **L** Liskov Substitution | Any `IDeckService` implementation is a valid drop-in for `GameService`. |
| **I** Interface Segregation | `IRoomManager`, `IGameService`, `IDeckService` are separate — consumers only take what they need. |
| **D** Dependency Inversion | `GameHub` depends on `IGameService` and `IRoomManager`, never on concrete classes. All registered in DI. |

---

## Phase 1 — Project Setup

### 1.1 Scaffold

```bash
dotnet new sln -n Overdraw
dotnet new webapp -n Overdraw.Web -o src/Overdraw.Web
dotnet new xunit -n Overdraw.Tests -o tests/Overdraw.Tests
dotnet sln add src/Overdraw.Web
dotnet sln add tests/Overdraw.Tests
dotnet add tests/Overdraw.Tests reference src/Overdraw.Web
```

### 1.2 Packages

```bash
# Overdraw.Web
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design

# Overdraw.Tests
dotnet add package Moq
dotnet add package FluentAssertions
```

### 1.3 Tailwind (v4)

Tailwind v4 is **CSS-first**: there is no `tailwind.config.js`, no `npx tailwindcss init`, and no `content` array — source files are **auto-detected**. The CLI also ships as its own package (`@tailwindcss/cli`); the main `tailwindcss` package no longer contains the CLI binary.

```bash
cd src/Overdraw.Web
npm init -y
npm install -D @tailwindcss/cli
```

`wwwroot/css/input.css` — a single import replaces the v3 `@tailwind base/components/utilities` directives:
```css
@import "tailwindcss";
```

**Source detection is automatic.** Running the CLI from `src/Overdraw.Web` scans `Pages/**/*.cshtml` and `wwwroot/js/**/*.js` with no configuration (gitignored files and `node_modules` are skipped). Only add an explicit `@source "..."` line in `input.css` to pull in paths *outside* the project.

`package.json` scripts (the `tailwindcss` binary resolves to `@tailwindcss/cli`):
```json
"scripts": {
  "css:watch": "tailwindcss -i ./wwwroot/css/input.css -o ./wwwroot/css/app.css --watch",
  "css:build": "tailwindcss -i ./wwwroot/css/input.css -o ./wwwroot/css/app.css --minify"
}
```

Add `wwwroot/css/app.css` to `.gitignore` — it's a build artifact.

### 1.4 Clean Bootstrap/jQuery

Remove from `wwwroot/lib/`:
- `bootstrap/`
- `jquery/`
- `jquery-validation/`
- `jquery-validation-unobtrusive/`

Remove references in `Pages/Shared/_Layout.cshtml` and delete `Pages/Shared/_ValidationScriptsPartial.cshtml`.

---

## Phase 2 — Models

```csharp
// Models/Card.cs
public record Card(string Suit, string Value)
{
    public int NumericValue => Value switch
    {
        "A"                  => 11,
        "J" or "Q" or "K"   => 10,
        _                    => int.Parse(Value)
    };
}

// Models/Hand.cs
public class Hand
{
    private readonly List<Card> _cards = new();
    public IReadOnlyList<Card> Cards => _cards.AsReadOnly();

    public void Add(Card card) => _cards.Add(card);

    public int Value
    {
        get
        {
            int total = _cards.Sum(c => c.NumericValue);
            int aces  = _cards.Count(c => c.Value == "A");
            while (total > 21 && aces > 0) { total -= 10; aces--; }
            return total;
        }
    }

    public bool IsBust      => Value > 21;
    public bool IsBlackjack => _cards.Count == 2 && Value == 21;
}

// Models/Enums/GameStatus.cs
public enum GameStatus { Lobby, Playing, DealerTurn, Finished }

// Models/Enums/PlayerStatus.cs
public enum PlayerStatus { Waiting, Playing, Standing, Bust, Blackjack }

// Models/Player.cs
public class Player
{
    public string ConnectionId { get; set; } = string.Empty;
    public string Name         { get; set; } = string.Empty;
    public Hand Hand           { get; set; } = new();
    public PlayerStatus Status { get; set; } = PlayerStatus.Waiting;
}

// Models/Game.cs
public class Game
{
    public string RoomCode           { get; set; } = string.Empty;
    public List<Player> Players      { get; set; } = new();
    public List<Card> Deck           { get; set; } = new();
    public Hand DealerHand           { get; set; } = new();
    public GameStatus Status         { get; set; } = GameStatus.Lobby;
    public int CurrentPlayerIndex    { get; set; } = 0;

    public Player? CurrentPlayer =>
        Status == GameStatus.Playing && CurrentPlayerIndex < Players.Count
            ? Players[CurrentPlayerIndex] : null;
}
```

---

## Phase 3 — Services

### Interfaces

```csharp
// Services/Interfaces/IDeckService.cs
public interface IDeckService
{
    List<Card> CreateShuffledDeck();
    Card Deal(List<Card> deck);
}

// Services/Interfaces/IRoomManager.cs
public interface IRoomManager
{
    Game GetOrCreate(string roomCode);
    Game? Get(string roomCode);
    void Remove(string roomCode);
}

// Services/Interfaces/IGameService.cs
public interface IGameService
{
    void AddPlayer(Game game, string connectionId, string name);
    void StartGame(Game game);
    void Hit(Game game, string connectionId);
    void Stand(Game game, string connectionId);
    bool IsPlayerTurn(Game game, string connectionId);
    GameResult GetResult(Game game);
}
```

### DeckService

```csharp
public class DeckService : IDeckService
{
    private static readonly string[] Suits  = { "♠","♥","♦","♣" };
    private static readonly string[] Values = { "2","3","4","5","6","7","8","9","10","J","Q","K","A" };

    public List<Card> CreateShuffledDeck() =>
        Suits.SelectMany(s => Values.Select(v => new Card(s, v)))
             .OrderBy(_ => Random.Shared.Next())
             .ToList();

    public Card Deal(List<Card> deck)
    {
        if (deck.Count == 0) throw new InvalidOperationException("Deck is empty.");
        var card = deck[0];
        deck.RemoveAt(0);
        return card;
    }
}
```

### RoomManager

```csharp
public class RoomManager : IRoomManager
{
    private readonly ConcurrentDictionary<string, Game> _rooms = new();

    public Game GetOrCreate(string roomCode) =>
        _rooms.GetOrAdd(roomCode, code => new Game { RoomCode = code });

    public Game? Get(string roomCode) =>
        _rooms.TryGetValue(roomCode, out var game) ? game : null;

    public void Remove(string roomCode) => _rooms.TryRemove(roomCode, out _);
}
```

### GameService

```csharp
public class GameService : IGameService
{
    private readonly IDeckService _deckService;
    public GameService(IDeckService deckService) => _deckService = deckService;

    public void AddPlayer(Game game, string connectionId, string name)
    {
        if (game.Players.Count >= 8)
            throw new InvalidOperationException("Table is full.");
        if (game.Status != GameStatus.Lobby)
            throw new InvalidOperationException("Game already in progress.");
        game.Players.Add(new Player { ConnectionId = connectionId, Name = name });
    }

    public void StartGame(Game game)
    {
        game.Deck       = _deckService.CreateShuffledDeck();
        game.DealerHand = new Hand();
        foreach (var p in game.Players) p.Hand = new Hand();

        for (int i = 0; i < 2; i++)
        {
            foreach (var p in game.Players) p.Hand.Add(_deckService.Deal(game.Deck));
            game.DealerHand.Add(_deckService.Deal(game.Deck));
        }

        game.Status = GameStatus.Playing;
        game.CurrentPlayerIndex = 0;

        foreach (var p in game.Players.Where(p => p.Hand.IsBlackjack))
            p.Status = PlayerStatus.Blackjack;

        AdvanceIfNeeded(game);
    }

    public void Hit(Game game, string connectionId)
    {
        var player = GetCurrentPlayer(game, connectionId);
        player.Hand.Add(_deckService.Deal(game.Deck));
        if (player.Hand.IsBust) player.Status = PlayerStatus.Bust;
        AdvanceIfNeeded(game);
    }

    public void Stand(Game game, string connectionId)
    {
        var player = GetCurrentPlayer(game, connectionId);
        player.Status = PlayerStatus.Standing;
        AdvanceIfNeeded(game);
    }

    public bool IsPlayerTurn(Game game, string connectionId) =>
        game.CurrentPlayer?.ConnectionId == connectionId;

    public GameResult GetResult(Game game)
    {
        int dealerValue = game.DealerHand.Value;
        var results = game.Players.Select(p => new PlayerResult(
            p.Name,
            p.Status switch
            {
                PlayerStatus.Bust                                        => "Bust",
                PlayerStatus.Blackjack when game.DealerHand.IsBlackjack => "Push",
                PlayerStatus.Blackjack                                   => "Blackjack!",
                _ when dealerValue > 21                                  => "Win",
                _ when p.Hand.Value > dealerValue                        => "Win",
                _ when p.Hand.Value == dealerValue                       => "Push",
                _                                                        => "Lose"
            }
        )).ToList();
        return new GameResult(results);
    }

    private void AdvanceIfNeeded(Game game)
    {
        while (game.CurrentPlayerIndex < game.Players.Count &&
               game.Players[game.CurrentPlayerIndex].Status is not
                   (PlayerStatus.Waiting or PlayerStatus.Playing))
        {
            game.CurrentPlayerIndex++;
        }
        if (game.CurrentPlayerIndex >= game.Players.Count)
            game.Status = GameStatus.DealerTurn;
    }

    private static Player GetCurrentPlayer(Game game, string connectionId)
    {
        var player = game.CurrentPlayer
            ?? throw new InvalidOperationException("No active player.");
        if (player.ConnectionId != connectionId)
            throw new InvalidOperationException("Not your turn.");
        player.Status = PlayerStatus.Playing;
        return player;
    }
}
```

---

## Phase 4 — SignalR Hub

```csharp
public class GameHub : Hub
{
    private readonly IGameService _gameService;
    private readonly IRoomManager _roomManager;

    public GameHub(IGameService gameService, IRoomManager roomManager)
    {
        _gameService = gameService;
        _roomManager = roomManager;
    }

    public async Task JoinRoom(string roomCode, string playerName)
    {
        var game = _roomManager.GetOrCreate(roomCode);
        _gameService.AddPlayer(game, Context.ConnectionId, playerName);
        await Groups.AddToGroupAsync(Context.ConnectionId, roomCode);
        await Clients.Group(roomCode).SendAsync("PlayerJoined", playerName, game.Players.Count);
    }

    public async Task StartGame(string roomCode)
    {
        var game = _roomManager.Get(roomCode) ?? throw new HubException("Room not found.");
        _gameService.StartGame(game);
        await Clients.Group(roomCode).SendAsync("GameStarted", new
        {
            Players      = game.Players.Select(p => new { p.Name, p.Status, Cards = p.Hand.Cards }),
            DealerCard   = game.DealerHand.Cards[0],
            CurrentPlayer = game.CurrentPlayer?.Name
        });
        if (game.Status == GameStatus.DealerTurn) await RunDealerTurn(roomCode, game);
    }

    public async Task Hit(string roomCode)
    {
        var game   = _roomManager.Get(roomCode) ?? throw new HubException("Room not found.");
        _gameService.Hit(game, Context.ConnectionId);
        var player = game.Players.First(p => p.ConnectionId == Context.ConnectionId);
        await Clients.Group(roomCode).SendAsync("CardDealt",
            player.Name, player.Hand.Cards.Last(), player.Status.ToString());
        await AdvanceTurn(roomCode, game);
    }

    public async Task Stand(string roomCode)
    {
        var game = _roomManager.Get(roomCode) ?? throw new HubException("Room not found.");
        _gameService.Stand(game, Context.ConnectionId);
        await AdvanceTurn(roomCode, game);
    }

    private async Task AdvanceTurn(string roomCode, Game game)
    {
        if (game.Status == GameStatus.DealerTurn)
            await RunDealerTurn(roomCode, game);
        else
            await Clients.Group(roomCode).SendAsync("TurnChanged", game.CurrentPlayer?.Name);
    }

    private async Task RunDealerTurn(string roomCode, Game game)
    {
        await Clients.Group(roomCode).SendAsync("DealerCardRevealed", game.DealerHand.Cards);
        while (game.DealerHand.Value < 17)
        {
            var card = game.Deck[0];
            game.Deck.RemoveAt(0);
            game.DealerHand.Add(card);
            await Clients.Group(roomCode).SendAsync("DealerHit", card, game.DealerHand.Value);
            await Task.Delay(800);
        }
        game.Status = GameStatus.Finished;
        await Clients.Group(roomCode).SendAsync("RoundOver", _gameService.GetResult(game));
    }
}
```

---

## Phase 5 — Data Layer

```csharp
// Data/Entities/GameRecord.cs
public class GameRecord
{
    public int Id           { get; set; }
    public string RoomCode  { get; set; } = string.Empty;
    public DateTime PlayedAt { get; set; }
    public int PlayerCount  { get; set; }
    public string ResultsJson { get; set; } = string.Empty;
}

// Data/AppDbContext.cs
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
    public DbSet<GameRecord> GameRecords => Set<GameRecord>();
}
```

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

---

## Phase 6 — Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorPages();
builder.Services.AddSignalR();

builder.Services.AddSingleton<IRoomManager, RoomManager>();  // shared state
builder.Services.AddScoped<IDeckService, DeckService>();
builder.Services.AddScoped<IGameService, GameService>();

builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlite(builder.Configuration.GetConnectionString("Default")
        ?? "Data Source=overdraw.db"));

var app = builder.Build();
app.UseStaticFiles();
app.UseRouting();
app.MapRazorPages();
app.MapHub<GameHub>("/gamehub");
app.Run();
```

`appsettings.json`:
```json
{
  "ConnectionStrings": {
    "Default": "Data Source=overdraw.db"
  }
}
```

---

## Phase 7 — SignalR JS Client (skeleton)

```js
// wwwroot/js/game.js
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/gamehub")
    .withAutomaticReconnect()
    .build();

connection.on("PlayerJoined",        (name, count)              => { });
connection.on("GameStarted",         (data)                     => { });
connection.on("CardDealt",           (player, card, status)     => { });
connection.on("TurnChanged",         (playerName)               => { });
connection.on("DealerCardRevealed",  (cards)                    => { });
connection.on("DealerHit",           (card, total)              => { });
connection.on("RoundOver",           (results)                  => { });

await connection.start();

const joinRoom  = (code, name) => connection.invoke("JoinRoom", code, name);
const startGame = (code)       => connection.invoke("StartGame", code);
const hit       = (code)       => connection.invoke("Hit", code);
const stand     = (code)       => connection.invoke("Stand", code);
```

---

## Phase 8 — Unit Tests

```csharp
public class GameServiceTests
{
    private readonly Mock<IDeckService> _deckMock = new();
    private IGameService Sut() => new GameService(_deckMock.Object);

    [Fact]
    public void StartGame_DealsEachPlayerTwoCards()
    {
        var deck = MakeDeck(20);
        _deckMock.Setup(d => d.CreateShuffledDeck()).Returns(deck);
        _deckMock.Setup(d => d.Deal(It.IsAny<List<Card>>()))
                 .Returns<List<Card>>(d => { var c = d[0]; d.RemoveAt(0); return c; });

        var game = new Game();
        var sut  = Sut();
        sut.AddPlayer(game, "conn1", "Alice");
        sut.StartGame(game);

        game.Players[0].Hand.Cards.Count.Should().Be(2);
    }

    [Fact]
    public void Hit_MarksPlayerBust_WhenHandExceeds21()
    {
        // arrange: player has 20, next card dealt is a 5
        // act: Hit
        // assert: player.Status == Bust
    }

    private static List<Card> MakeDeck(int n) =>
        Enumerable.Range(0, n).Select(_ => new Card("♠", "5")).ToList();
}
```

---

## Checklist

- [ ] Solution + projects scaffolded
- [ ] Bootstrap/jQuery removed, Tailwind wired
- [ ] Models — Card, Hand, Player, Game, Enums
- [ ] DeckService + unit tests
- [ ] GameService + unit tests (hit, stand, bust, blackjack, dealer)
- [ ] RoomManager
- [ ] DI wired in Program.cs
- [ ] GameHub — all methods + broadcasts
- [ ] Razor Pages — Index + Game
- [ ] game.js — all SignalR handlers
- [ ] EF Core migration + GameRecord
- [ ] Deploy to Railway/Render
- [ ] Linked on jaridijk.nl

---

*Last updated: August 2026*
