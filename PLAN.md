# Blindy2 - Discord Blind Test Bot - Complete Plan

## Table of Contents
1. [Overview](#overview)
2. [How Discord Bots Work](#how-discord-bots-work)
3. [Game Mechanics](#game-mechanics)
4. [Answer Validation System](#answer-validation-system)
5. [Project Architecture](#project-architecture)
6. [Detailed Component Breakdown](#detailed-component-breakdown)
7. [Commands Reference](#commands-reference)
8. [Data Storage](#data-storage)
9. [Game Flow Examples](#game-flow-examples)
10. [Implementation Files](#implementation-files)
11. [Setup Process](#setup-process)

---

## Overview

**Blindy2** is a Discord bot that helps you run blind test (music quiz) games in your Discord server.

### What it does:
- You (the admin) play music manually (Spotify, YouTube, etc.)
- Players type their guesses in a dedicated Discord channel
- The bot validates answers and awards points
- First correct answer locks the question (prevents spam)
- Tracks scores and shows a leaderboard

### What it doesn't do:
- It does NOT play music automatically
- It does NOT search for songs
- It does NOT host multiple games simultaneously

---

## How Discord Bots Work

### Basic Concept
A Discord bot is just a **Python program that runs on your computer** (or a server) and connects to Discord's servers through the internet.

```
Your Computer                    Discord's Servers                Your Server
┌─────────────┐                 ┌──────────────┐                ┌─────────────┐
│             │                 │              │                │             │
│  bot.py     │◄───Internet────►│   Discord    │◄───Internet───►│  Players    │
│  (Python)   │                 │   API        │                │  typing     │
│             │                 │              │                │  messages   │
└─────────────┘                 └──────────────┘                └─────────────┘
```

### Key Components

1. **Bot Token**: A secret password that lets your Python program log in as the bot
2. **Intents**: Permissions that tell Discord what your bot needs to see (messages, users, etc.)
3. **Events**: Things that happen in Discord (new message, user joins, etc.)
4. **Commands**: Special messages that trigger bot actions (e.g., `/start_round`)

### How Messages Work

When someone types in Discord:
1. Discord receives the message
2. Discord sends it to your bot (because you enabled "Message Content Intent")
3. Your Python code reads the message
4. Your code decides what to do (validate answer, ignore it, etc.)
5. Your code sends a response back to Discord
6. Discord shows your bot's response to everyone

### Two Types of Commands

**Slash Commands** (modern):
- Type `/` and see a menu of commands
- Discord shows you what parameters are needed
- Example: `/start_round artist:queen title:bohemian rhapsody`
- Used for: Admin commands, user commands

**Regular Messages** (traditional):
- Just type normally in chat
- Example: `queen`
- Used for: Player guesses (faster, more natural)

---

## Game Mechanics

### Core Rules

1. **Admin starts a round** by typing `/start_round` with the correct answer
2. **Players type guesses** as regular messages in the game channel
3. **First correct answer wins** and locks the question
4. **Partial answers count**: Guessing only the artist OR only the title = 1 point
5. **Full answers count more**: Guessing both artist AND title = 2 points
6. **Admin ends the round** when ready to move on

### Scoring System

| What Player Types | Points Awarded | Example |
|-------------------|----------------|---------|
| Only artist name | 1 point | `queen` |
| Only song title | 1 point | `bohemian rhapsody` |
| Both (any order) | 2 points | `queen bohemian rhapsody` or `bohemian rhapsody queen` |

### Locking Mechanism

**Why?** To prevent spam after someone guesses correctly.

**How it works:**
1. Round starts → unlocked state
2. First person guesses correctly → **LOCKED**
3. Everyone else's messages after that → ignored (no points, no response)
4. Admin ends round → reset for next round

**Example Timeline:**
```
[10:00:00] Admin: /start_round artist:queen title:bohemian rhapsody
[10:00:01] Bot: 🎵 Round 1 started! Start guessing!
[10:00:15] Player1: queen
[10:00:15] Bot: ✅ Player1 got the artist! (+1 point) 🔒 Question locked!
[10:00:16] Player2: bohemian rhapsody    ← Too late! No response from bot
[10:00:17] Player3: queen                ← Too late! No response from bot
[10:01:00] Admin: /end_round
[10:01:00] Bot: 📊 Round 1 ended! Answer: queen - bohemian rhapsody
                  Winner: Player1 (+1 point)
```

### Channel Setup

**Game Channel** (one specific channel):
- Bot listens for guesses here
- Bot announces round start/end here
- Example: `#blind-test`

**Admin Commands** (work from any channel):
- Admin can type `/start_round` from `#admin-chat`
- Admin can type `/end_round` from `#music-requests`
- Makes it easier to control without cluttering game channel

---

## Answer Validation System

### The Problem
If answers are too loose, people can spam random words. If answers are too strict, valid guesses might be rejected.

### Our Solution: STRICT Mode

Players must type answers with:
- ✅ **Lowercase only**: "queen" accepted, "Queen" rejected
- ✅ **Exact match**: Message must be ONLY the answer, nothing else
- ✅ **No punctuation**: "queen!" rejected
- ✅ **No extra words**: "i think queen" rejected
- ✅ **Flexible word order for both**: "queen bohemian rhapsody" OR "bohemian rhapsody queen" both work

### Validation Algorithm (Step-by-Step)

When a player types a message in the game channel:

```
Step 1: Is the round active and unlocked?
├─ No → Ignore message silently
└─ Yes → Continue to Step 2

Step 2: Normalize whitespace
├─ "queen    bohemian  rhapsody" → "queen bohemian rhapsody"
└─ "  queen  " → "queen"

Step 3: Does the message contain ANY uppercase letters?
├─ Yes (e.g., "Queen") → Reject silently (no response)
└─ No → Continue to Step 4

Step 4: Convert everything to lowercase for comparison
├─ Player's message → lowercase
├─ Correct artist → lowercase
└─ Correct title → lowercase

Step 5: Check for exact matches
├─ Does message exactly equal artist? → Award 1 point, lock question
├─ Does message exactly equal title? → Award 1 point, lock question
├─ Does message equal "artist title"? → Award 2 points, lock question
├─ Does message equal "title artist"? → Award 2 points, lock question
└─ None of the above? → Reject silently (no response)
```

### Validation Examples

Let's say the correct answer is:
- Artist: `queen`
- Title: `bohemian rhapsody`

| Player Types | Validation Result | Points | Why |
|--------------|-------------------|--------|-----|
| `queen` | ✅ Accepted | 1 | Exact match for artist |
| `bohemian rhapsody` | ✅ Accepted | 1 | Exact match for title |
| `queen bohemian rhapsody` | ✅ Accepted | 2 | Both in one message |
| `bohemian rhapsody queen` | ✅ Accepted | 2 | Both in one message (order flexible) |
| `Queen` | ❌ Rejected | 0 | Contains uppercase |
| `QUEEN` | ❌ Rejected | 0 | Contains uppercase |
| `queen!` | ❌ Rejected | 0 | Contains punctuation (not exact match) |
| `i think queen` | ❌ Rejected | 0 | Contains extra words (not exact match) |
| `queen?` | ❌ Rejected | 0 | Contains punctuation |
| `the queen` | ❌ Rejected | 0 | Contains extra word |
| `bohemian` | ❌ Rejected | 0 | Incomplete (not exact match) |
| `queen  bohemian  rhapsody` | ✅ Accepted | 2 | Whitespace normalized to "queen bohemian rhapsody" |

### Why Silent Rejection?

When someone types an invalid guess (uppercase, extra words, etc.), the bot does **nothing**. No response at all.

**Why?**
- Prevents chat spam (imagine 20 people getting "Wrong format!" messages)
- Keeps the game moving fast
- Players learn the format quickly
- Clean chat history

---

## Project Architecture

### Folder Structure

```
blind_bot/
├── bot.py                          # Main entry point (starts the bot)
├── cogs/                            # Command modules (organized features)
│   ├── __init__.py                 # Makes "cogs" a Python package
│   ├── game.py                     # Game logic (guesses, scores, /current)
│   └── admin.py                    # Admin commands (/start_round, /end_round)
├── utils/                           # Helper functions (reusable code)
│   ├── __init__.py                 # Makes "utils" a Python package
│   ├── answer_checker.py           # Answer validation logic
│   ├── data_manager.py             # Reading/writing JSON files
│   └── checks.py                   # Permission checking decorators
├── data/                            # Persistent storage
│   ├── config.json                 # Bot settings (game channel ID)
│   └── scores.json                 # Player scores (persistent)
├── .env                             # SECRET: Bot token (never commit to git!)
├── .env.example                     # Template for .env file
├── .gitignore                       # Tells git what files to ignore
├── requirements.txt                 # Python dependencies (discord.py, etc.)
└── README.md                        # Setup instructions
```

### What is a "Cog"?

In discord.py, a **Cog** is a way to organize related commands into separate files.

**Without Cogs** (everything in one file):
```python
# bot.py - 1000 lines of messy code
@bot.command()
async def start_round(...):
    # code here

@bot.command()
async def end_round(...):
    # code here

@bot.command()
async def scores(...):
    # code here
# ... 50 more commands ...
```

**With Cogs** (organized):
```python
# bot.py - clean and simple
bot.load_extension('cogs.admin')
bot.load_extension('cogs.game')

# cogs/admin.py - admin commands only
class AdminCog(commands.Cog):
    @app_commands.command()
    async def start_round(...):
        # code here

# cogs/game.py - game commands only
class GameCog(commands.Cog):
    @app_commands.command()
    async def scores(...):
        # code here
```

**Benefits:**
- Easier to find code
- Each file has a clear purpose
- Can enable/disable features by (un)loading cogs
- Multiple people can work on different cogs simultaneously

---

## Detailed Component Breakdown

### 1. bot.py - The Main Entry Point

**Purpose:** Start the bot and load all extensions.

**What it does:**
1. Load the bot token from `.env` file
2. Create a Discord bot client
3. Enable required "intents" (permissions to see messages)
4. Load the two cogs (admin.py and game.py)
5. Sync slash commands with Discord (so `/start_round` appears in the menu)
6. Print "Blindy2 is online!" when ready

**Code structure:**
```python
import discord
from discord.ext import commands
import os
from dotenv import load_dotenv

# Load token from .env
load_dotenv()
TOKEN = os.getenv('DISCORD_TOKEN')

# Create bot with required intents
intents = discord.Intents.default()
intents.message_content = True  # CRITICAL: Lets bot read messages
intents.guilds = True           # Lets bot see server info

bot = commands.Bot(command_prefix='!', intents=intents)

# When bot is ready
@bot.event
async def on_ready():
    print(f'{bot.user.name} is online!')
    # Sync slash commands
    await bot.tree.sync()

# Load cogs
async def load_extensions():
    await bot.load_extension('cogs.admin')
    await bot.load_extension('cogs.game')

# Start the bot
async def main():
    await load_extensions()
    await bot.start(TOKEN)

# Run
import asyncio
asyncio.run(main())
```

**Key Concepts:**
- `load_dotenv()`: Reads `.env` file and loads `DISCORD_TOKEN` variable
- `intents`: Tell Discord what events your bot needs to receive
- `message_content = True`: Required to read what users type (privacy setting)
- `bot.tree.sync()`: Uploads slash commands to Discord's servers
- `asyncio`: Python's way of handling multiple things at once (needed for bots)

---

### 2. utils/answer_checker.py - Answer Validation

**Purpose:** Determine if a player's guess is correct and how many points to award.

**Input:**
- Player's message content (e.g., "queen bohemian rhapsody")
- Correct artist (e.g., "queen")
- Correct title (e.g., "bohemian rhapsody")

**Output:**
- Tuple: `(is_correct: bool, points: int, match_type: str or None)`
- Examples:
  - `(True, 2, "both")` → Player got both
  - `(True, 1, "artist")` → Player got artist only
  - `(True, 1, "title")` → Player got title only
  - `(False, 0, None)` → Wrong answer

**Algorithm:**
```python
def check_answer(message: str, artist: str, title: str) -> tuple:
    # Step 1: Normalize whitespace
    guess = ' '.join(message.strip().split())
    artist_normalized = ' '.join(artist.strip().split())
    title_normalized = ' '.join(title.strip().split())
    
    # Step 2: Check for uppercase (reject if found)
    if guess != guess.lower():
        return (False, 0, None)
    
    # Step 3: Convert to lowercase for comparison
    guess = guess.lower()
    artist_lower = artist_normalized.lower()
    title_lower = title_normalized.lower()
    
    # Step 4: Check exact matches
    both_option1 = f"{artist_lower} {title_lower}"
    both_option2 = f"{title_lower} {artist_lower}"
    
    if guess == both_option1 or guess == both_option2:
        return (True, 2, "both")
    elif guess == artist_lower:
        return (True, 1, "artist")
    elif guess == title_lower:
        return (True, 1, "title")
    else:
        return (False, 0, None)
```

**Example Executions:**

```python
# Correct answer: artist="Queen", title="Bohemian Rhapsody"

check_answer("queen", "Queen", "Bohemian Rhapsody")
# Returns: (True, 1, "artist")

check_answer("Queen", "Queen", "Bohemian Rhapsody")
# Returns: (False, 0, None)  ← Uppercase rejected

check_answer("queen bohemian rhapsody", "Queen", "Bohemian Rhapsody")
# Returns: (True, 2, "both")

check_answer("bohemian rhapsody queen", "Queen", "Bohemian Rhapsody")
# Returns: (True, 2, "both")  ← Order doesn't matter

check_answer("queen!", "Queen", "Bohemian Rhapsody")
# Returns: (False, 0, None)  ← Punctuation makes it not exact match
```

---

### 3. utils/data_manager.py - JSON File Operations

**Purpose:** Save and load data (scores, config) from JSON files.

**Why JSON?**
- Human-readable text format
- Easy to edit manually if needed
- Python has built-in JSON support
- Good for small to medium datasets

**What it manages:**

**config.json:**
```json
{
  "game_channel_id": 123456789012345678,
  "admin_permission": "manage_channels"
}
```

**scores.json:**
```json
{
  "234567890123456789": {
    "username": "Player1",
    "total_points": 15,
    "rounds_won": 8,
    "full_answers": 5,
    "partial_answers": 3
  },
  "345678901234567890": {
    "username": "Player2",
    "total_points": 12,
    "rounds_won": 6,
    "full_answers": 4,
    "partial_answers": 2
  }
}
```

**Key Functions:**

```python
class DataManager:
    def load_config(self) -> dict:
        """Load config.json or return defaults if missing"""
        
    def save_config(self, config: dict):
        """Save config.json atomically (safe write)"""
        
    def load_scores(self) -> dict:
        """Load scores.json or return empty dict if missing"""
        
    def save_scores(self, scores: dict):
        """Save scores.json atomically"""
        
    def add_score(self, user_id: str, username: str, points: int, match_type: str):
        """
        Add points to a user's score
        - Load current scores
        - Update user's entry (or create if new)
        - Increment counters
        - Save back to file
        """
        
    def get_leaderboard(self, limit: int = 10) -> list:
        """
        Get top N players sorted by score
        - Load scores
        - Sort by: total_points (descending), then username (ascending)
        - Return top N entries
        """
```

**Atomic Writes** (important for data safety):

When saving a file, don't write directly to `scores.json`. Instead:
1. Write to `scores.json.tmp` (temporary file)
2. If write succeeds, rename `scores.json.tmp` → `scores.json`
3. If write fails, `scores.json` is still intact (not corrupted)

This prevents data loss if the program crashes during a save.

---

### 4. utils/checks.py - Permission Checking

**Purpose:** Ensure only admins can use admin commands.

**Discord Permissions:**
Discord has built-in permissions like:
- `manage_channels` - Can create/edit/delete channels
- `manage_messages` - Can delete others' messages
- `manage_server` - Can change server settings
- `administrator` - Can do everything

We chose `manage_channels` as the requirement for admin commands.

**How it works:**

```python
from discord import app_commands, Interaction

async def has_manage_channels(interaction: Interaction) -> bool:
    """Check if user has manage_channels permission"""
    # Check if user has the permission
    if interaction.user.guild_permissions.manage_channels:
        return True
    else:
        # Send error message (only visible to that user)
        await interaction.response.send_message(
            "❌ You need 'Manage Channels' permission to use this command.",
            ephemeral=True  # Only the command user sees this
        )
        return False
```

**Usage in commands:**

```python
@app_commands.command()
@app_commands.check(has_manage_channels)
async def start_round(interaction, artist: str, title: str):
    # This code only runs if user has manage_channels permission
    ...
```

---

### 5. cogs/game.py - Game Logic & User Commands

**Purpose:** Handle the game state, process guesses, provide user commands.

**Game State Variables** (stored in memory, lost on restart):

```python
class GameCog(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.data_manager = DataManager()
        
        # Game state
        self.active = False              # Is a round currently running?
        self.artist = None               # Correct artist (lowercase)
        self.title = None                # Correct title (lowercase)
        self.locked = False              # Has someone answered correctly?
        self.winner_id = None            # Discord user ID of winner
        self.winner_name = None          # Display name of winner
        self.points_awarded = 0          # How many points awarded (1 or 2)
        self.started_at = None           # When round started (for elapsed time)
        self.round_number = 0            # Current round number
        self.game_channel_id = None      # Where to listen for guesses
        
        # Load game channel from config
        config = self.data_manager.load_config()
        self.game_channel_id = config.get('game_channel_id')
```

**Message Listener** (processes guesses):

```python
@commands.Cog.listener()
async def on_message(self, message):
    """Called every time someone sends a message in Discord"""
    
    # Ignore messages from bots (including ourselves)
    if message.author.bot:
        return
    
    # Only process messages in the game channel
    if message.channel.id != self.game_channel_id:
        return
    
    # Only process if round is active and not locked
    if not self.active or self.locked:
        return
    
    # Validate the answer
    is_correct, points, match_type = check_answer(
        message.content,
        self.artist,
        self.title
    )
    
    # If correct, lock the round and award points
    if is_correct:
        self.locked = True
        self.winner_id = message.author.id
        self.winner_name = message.author.display_name
        self.points_awarded = points
        
        # Save to persistent storage
        self.data_manager.add_score(
            str(message.author.id),
            message.author.display_name,
            points,
            match_type
        )
        
        # Announce the win
        if match_type == "both":
            await message.channel.send(
                f"✅ {message.author.display_name} got both! (+{points} points)\n"
                f"🔒 Question locked!"
            )
        elif match_type == "artist":
            await message.channel.send(
                f"✅ {message.author.display_name} got the artist! (+{points} point)\n"
                f"🔒 Question locked!"
            )
        elif match_type == "title":
            await message.channel.send(
                f"✅ {message.author.display_name} got the title! (+{points} point)\n"
                f"🔒 Question locked!"
            )
    
    # If incorrect, do nothing (silent rejection)
```

**User Commands:**

```python
@app_commands.command(name="scores", description="Show top 10 leaderboard")
async def scores(self, interaction: Interaction):
    """Display the leaderboard"""
    leaderboard = self.data_manager.get_leaderboard(limit=10)
    
    if not leaderboard:
        await interaction.response.send_message("No scores yet!")
        return
    
    # Create a nice embed (colored box with formatted text)
    embed = discord.Embed(
        title="🏆 Top 10 Leaderboard",
        color=0xFFD700  # Gold color
    )
    
    for rank, (user_id, data) in enumerate(leaderboard, start=1):
        embed.add_field(
            name=f"{rank}. {data['username']}",
            value=f"{data['total_points']} points ({data['rounds_won']} wins)",
            inline=False
        )
    
    await interaction.response.send_message(embed=embed)

@app_commands.command(name="current", description="Show current round status")
async def current(self, interaction: Interaction):
    """Show info about the active round"""
    if not self.active:
        await interaction.response.send_message("No active round.")
        return
    
    # Calculate elapsed time
    elapsed = datetime.now() - self.started_at
    elapsed_str = str(elapsed).split('.')[0]  # Remove microseconds
    
    # Format status
    status = "🔒 Locked" if self.locked else "🟢 Active"
    
    embed = discord.Embed(
        title=f"Round {self.round_number}",
        color=0x00FF00  # Green
    )
    embed.add_field(name="Status", value=status, inline=True)
    embed.add_field(name="Time Elapsed", value=elapsed_str, inline=True)
    
    if self.locked:
        embed.add_field(
            name="Winner",
            value=f"{self.winner_name} (+{self.points_awarded} points)",
            inline=False
        )
    
    await interaction.response.send_message(embed=embed)
```

---

### 6. cogs/admin.py - Admin Commands

**Purpose:** Provide game control commands for admins.

**All commands use the permission check:**

```python
from utils.checks import has_manage_channels

class AdminCog(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.data_manager = DataManager()
```

**Command: /set_channel**

```python
@app_commands.command(name="set_channel", description="Set current channel as game channel")
@app_commands.check(has_manage_channels)
async def set_channel(self, interaction: Interaction):
    """Configure which channel to use for the game"""
    
    # Get current channel ID
    channel_id = interaction.channel.id
    
    # Save to config
    config = self.data_manager.load_config()
    config['game_channel_id'] = channel_id
    self.data_manager.save_config(config)
    
    # Update the game cog's channel ID
    game_cog = self.bot.get_cog('GameCog')
    game_cog.game_channel_id = channel_id
    
    # Confirm (ephemeral = only command user sees this)
    await interaction.response.send_message(
        f"✅ Game channel set to {interaction.channel.mention}",
        ephemeral=True
    )
```

**Command: /start_round**

```python
@app_commands.command(name="start_round", description="Start a new blind test round")
@app_commands.describe(
    artist="Artist name (lowercase)",
    title="Song title (lowercase)"
)
@app_commands.check(has_manage_channels)
async def start_round(self, interaction: Interaction, artist: str, title: str):
    """Start a new round with the given answer"""
    
    game_cog = self.bot.get_cog('GameCog')
    
    # Validation: Check if game channel is set
    if game_cog.game_channel_id is None:
        await interaction.response.send_message(
            "❌ Game channel not set! Use `/set_channel` first.",
            ephemeral=True
        )
        return
    
    # Validation: Check if round already active
    if game_cog.active:
        await interaction.response.send_message(
            "❌ Round already active! Use `/end_round` first.",
            ephemeral=True
        )
        return
    
    # Normalize inputs (remove extra whitespace, convert to lowercase)
    artist = ' '.join(artist.strip().split()).lower()
    title = ' '.join(title.strip().split()).lower()
    
    # Validation: Check for empty strings
    if not artist or not title:
        await interaction.response.send_message(
            "❌ Artist and title cannot be empty.",
            ephemeral=True
        )
        return
    
    # Start the round
    game_cog.active = True
    game_cog.artist = artist
    game_cog.title = title
    game_cog.locked = False
    game_cog.winner_id = None
    game_cog.winner_name = None
    game_cog.points_awarded = 0
    game_cog.started_at = datetime.now()
    game_cog.round_number += 1
    
    # Announce in game channel
    game_channel = self.bot.get_channel(game_cog.game_channel_id)
    await game_channel.send(
        f"🎵 **Round {game_cog.round_number} started!** Start guessing!"
    )
    
    # Confirm to admin (only admin sees this)
    await interaction.response.send_message(
        f"✅ Round {game_cog.round_number} started!\n"
        f"Answer: {artist} - {title}",
        ephemeral=True
    )
```

**Command: /end_round**

```python
@app_commands.command(name="end_round", description="End the current round")
@app_commands.check(has_manage_channels)
async def end_round(self, interaction: Interaction):
    """End the active round and show results"""
    
    game_cog = self.bot.get_cog('GameCog')
    
    # Validation: Check if round is active
    if not game_cog.active:
        await interaction.response.send_message(
            "❌ No active round.",
            ephemeral=True
        )
        return
    
    # Gather round info
    round_num = game_cog.round_number
    answer = f"{game_cog.artist} - {game_cog.title}"
    
    # End the round
    game_cog.active = False
    
    # Announce in game channel
    game_channel = self.bot.get_channel(game_cog.game_channel_id)
    
    if game_cog.locked:
        # Someone won
        await game_channel.send(
            f"📊 **Round {round_num} ended!**\n"
            f"Answer: {answer}\n"
            f"Winner: {game_cog.winner_name} (+{game_cog.points_awarded} points)"
        )
    else:
        # No one won
        await game_channel.send(
            f"📊 **Round {round_num} ended!**\n"
            f"Answer: {answer}\n"
            f"No one guessed correctly."
        )
    
    # Confirm to admin
    await interaction.response.send_message(
        "✅ Round ended.",
        ephemeral=True
    )
```

**Command: /set_answer**

```python
@app_commands.command(name="set_answer", description="Correct the answer if you made a typo")
@app_commands.describe(
    artist="Correct artist name",
    title="Correct song title"
)
@app_commands.check(has_manage_channels)
async def set_answer(self, interaction: Interaction, artist: str, title: str):
    """Update the answer mid-round"""
    
    game_cog = self.bot.get_cog('GameCog')
    
    if not game_cog.active:
        await interaction.response.send_message(
            "❌ No active round.",
            ephemeral=True
        )
        return
    
    # Normalize
    artist = ' '.join(artist.strip().split()).lower()
    title = ' '.join(title.strip().split()).lower()
    
    # Update
    game_cog.artist = artist
    game_cog.title = title
    
    await interaction.response.send_message(
        f"✅ Answer updated to: {artist} - {title}",
        ephemeral=True
    )
```

**Command: /show_answer**

```python
@app_commands.command(name="show_answer", description="Reveal the answer without ending")
@app_commands.check(has_manage_channels)
async def show_answer(self, interaction: Interaction):
    """Show the answer in the game channel (for skipping)"""
    
    game_cog = self.bot.get_cog('GameCog')
    
    if not game_cog.active:
        await interaction.response.send_message(
            "❌ No active round.",
            ephemeral=True
        )
        return
    
    answer = f"{game_cog.artist} - {game_cog.title}"
    
    game_channel = self.bot.get_channel(game_cog.game_channel_id)
    await game_channel.send(f"💡 Answer: {answer}")
    
    await interaction.response.send_message(
        "✅ Answer revealed.",
        ephemeral=True
    )
```

**Command: /reset_scores**

```python
@app_commands.command(name="reset_scores", description="Reset scores")
@app_commands.describe(user="User to reset (leave empty for all)")
@app_commands.check(has_manage_channels)
async def reset_scores(self, interaction: Interaction, user: Optional[discord.User] = None):
    """Reset all scores or a specific user's score"""
    
    if user:
        # Reset specific user
        scores = self.data_manager.load_scores()
        user_id = str(user.id)
        
        if user_id in scores:
            del scores[user_id]
            self.data_manager.save_scores(scores)
            await interaction.response.send_message(
                f"✅ Reset scores for {user.display_name}",
                ephemeral=True
            )
        else:
            await interaction.response.send_message(
                f"❌ No scores found for {user.display_name}",
                ephemeral=True
            )
    else:
        # Reset all scores
        self.data_manager.save_scores({})
        await interaction.response.send_message(
            "✅ All scores reset!",
            ephemeral=True
        )
```

---

## Commands Reference

### Admin Commands (Require "Manage Channels" Permission)

| Command | Parameters | Description | Example |
|---------|-----------|-------------|---------|
| `/set_channel` | None | Set current channel as game channel | `/set_channel` |
| `/start_round` | `artist`, `title` | Start a new round | `/start_round artist:queen title:bohemian rhapsody` |
| `/end_round` | None | End current round | `/end_round` |
| `/set_answer` | `artist`, `title` | Fix answer if you made a typo | `/set_answer artist:queen title:bohemian rhapsody` |
| `/show_answer` | None | Reveal answer without ending | `/show_answer` |
| `/reset_scores` | `user` (optional) | Reset all or specific user's scores | `/reset_scores` or `/reset_scores @Player1` |

### User Commands (Available to Everyone)

| Command | Description | Example |
|---------|-------------|---------|
| `/scores` | View top 10 leaderboard | `/scores` |
| `/current` | Show current round status | `/current` |

### Guessing (Regular Messages)

Just type your guess in the game channel!

**Rules:**
- Must be lowercase only
- Must be exact match (no extra words)
- Can guess artist, title, or both
- First correct answer wins

**Examples:**
- `queen` → 1 point if correct artist
- `bohemian rhapsody` → 1 point if correct title
- `queen bohemian rhapsody` → 2 points if both correct

---

## Data Storage

### config.json

**Location:** `data/config.json`

**Purpose:** Store bot settings that persist across restarts.

**Structure:**
```json
{
  "game_channel_id": 123456789012345678,
  "admin_permission": "manage_channels"
}
```

**Fields:**
- `game_channel_id`: Discord channel ID where guesses are processed (set via `/set_channel`)
- `admin_permission`: Which permission is required for admin commands (currently hardcoded to "manage_channels")

**When it's modified:**
- When admin uses `/set_channel`

**Initial state:**
```json
{
  "game_channel_id": null,
  "admin_permission": "manage_channels"
}
```

---

### scores.json

**Location:** `data/scores.json`

**Purpose:** Store player scores permanently (survives bot restarts).

**Structure:**
```json
{
  "user_id_1": {
    "username": "Player1",
    "total_points": 15,
    "rounds_won": 8,
    "full_answers": 5,
    "partial_answers": 3
  },
  "user_id_2": {
    "username": "Player2",
    "total_points": 12,
    "rounds_won": 6,
    "full_answers": 4,
    "partial_answers": 2
  }
}
```

**Fields per user:**
- `username`: Player's current display name in the server (updated each time they score)
- `total_points`: Total points accumulated
- `rounds_won`: Number of times they answered correctly (any points)
- `full_answers`: Number of 2-point answers (both artist and title)
- `partial_answers`: Number of 1-point answers (artist or title only)

**When it's modified:**
- Every time someone answers correctly (immediately after lock)
- When admin uses `/reset_scores`

**Initial state:**
```json
{}
```

**Why use Discord user ID as key?**
- User IDs never change (usernames and display names can)
- Guaranteed unique (no two users have same ID)
- Allows tracking even if user changes their name

---

### Game State (In-Memory Only)

**Location:** `cogs/game.py` → `GameCog` instance variables

**Purpose:** Track the current round's state.

**NOT saved to disk** → Lost when bot restarts.

**Variables:**
```python
self.active = False              # Is round running?
self.artist = None               # Correct artist (lowercase)
self.title = None                # Correct title (lowercase)
self.locked = False              # Has someone answered?
self.winner_id = None            # Winner's Discord user ID
self.winner_name = None          # Winner's display name
self.points_awarded = 0          # Points given to winner (1 or 2)
self.started_at = None           # When round started (datetime object)
self.round_number = 0            # Current round number
self.game_channel_id = None      # Channel ID (loaded from config)
```

**Lifecycle:**
1. Bot starts → All variables at default (False/None/0)
2. `/start_round` → Set active=True, artist, title, started_at, increment round_number
3. Player guesses correctly → Set locked=True, winner info, points_awarded
4. `/end_round` → Set active=False (other variables stay until next round)
5. Bot restarts → All state lost (scores.json and config.json are preserved)

---

## Game Flow Examples

### Example 1: Complete Round (Winner)

```
[Admin in #admin-chat]
/set_channel

[Bot response to Admin only]
✅ Game channel set to #blind-test

[Admin in #music-room]
/start_round artist:queen title:bohemian rhapsody

[Bot in #blind-test]
🎵 Round 1 started! Start guessing!

[Bot response to Admin only]
✅ Round 1 started!
Answer: queen - bohemian rhapsody

[Player1 in #blind-test]
the beatles

[No response from bot - wrong answer, silent rejection]

[Player2 in #blind-test]
Queen

[No response from bot - uppercase rejected silently]

[Player3 in #blind-test]
queen

[Bot in #blind-test]
✅ Player3 got the artist! (+1 point)
🔒 Question locked!

[Player4 in #blind-test]
bohemian rhapsody

[No response from bot - too late, question locked]

[Admin in #admin-chat]
/end_round

[Bot in #blind-test]
📊 Round 1 ended!
Answer: queen - bohemian rhapsody
Winner: Player3 (+1 point)

[Bot response to Admin only]
✅ Round ended.
```

**What happened:**
1. Admin set the game channel
2. Admin started round with correct answer
3. Player1 guessed wrong → silent
4. Player2 guessed uppercase → silent
5. Player3 guessed correct (artist only) → 1 point, locked
6. Player4 guessed after lock → silent
7. Admin ended round → summary shown

---

### Example 2: Complete Round (Full Answer)

```
[Admin]
/start_round artist:the beatles title:hey jude

[Bot in #blind-test]
🎵 Round 2 started! Start guessing!

[Player1 in #blind-test]
hey jude the beatles

[Bot in #blind-test]
✅ Player1 got both! (+2 points)
🔒 Question locked!

[Admin]
/end_round

[Bot in #blind-test]
📊 Round 2 ended!
Answer: the beatles - hey jude
Winner: Player1 (+2 points)
```

**What happened:**
- Player1 guessed both artist and title in one message (order doesn't matter)
- Awarded 2 points
- Question locked immediately

---

### Example 3: No Winner

```
[Admin]
/start_round artist:pink floyd title:comfortably numb

[Bot in #blind-test]
🎵 Round 3 started! Start guessing!

[Player1 in #blind-test]
led zeppelin

[No response - wrong, silent]

[Player2 in #blind-test]
stairway to heaven

[No response - wrong, silent]

[Admin waits 30 seconds, no one guesses]

[Admin]
/show_answer

[Bot in #blind-test]
💡 Answer: pink floyd - comfortably numb

[Admin]
/end_round

[Bot in #blind-test]
📊 Round 3 ended!
Answer: pink floyd - comfortably numb
No one guessed correctly.
```

**What happened:**
- No one guessed correctly
- Admin used `/show_answer` to reveal (optional)
- Admin ended round manually
- No points awarded

---

### Example 4: Admin Fixes Typo

```
[Admin]
/start_round artist:queen title:bohemain rhapsody

[Bot in #blind-test]
🎵 Round 4 started! Start guessing!

[Admin realizes typo: "bohemain" should be "bohemian"]

[Admin]
/set_answer artist:queen title:bohemian rhapsody

[Bot response to Admin only]
✅ Answer updated to: queen - bohemian rhapsody

[Player1 in #blind-test]
bohemian rhapsody

[Bot in #blind-test]
✅ Player1 got the title! (+1 point)
🔒 Question locked!
```

**What happened:**
- Admin made typo when starting round
- Admin fixed it with `/set_answer` before anyone guessed
- Player's guess validated against corrected answer

---

### Example 5: Multiple Players Racing

```
[Admin]
/start_round artist:nirvana title:smells like teen spirit

[Bot in #blind-test]
🎵 Round 5 started! Start guessing!

[10:00:00.123] Player1: nirvana
[10:00:00.456] Player2: nirvana
[10:00:00.789] Player3: smells like teen spirit

[Bot in #blind-test]
✅ Player1 got the artist! (+1 point)
🔒 Question locked!
```

**What happened:**
- Three players typed at almost the same time
- Discord's message timestamps determine true order
- Player1's message arrived first (10:00:00.123)
- Only Player1 gets points
- Player2 and Player3 get no response (too late)

---

### Example 6: Checking Scores

```
[Player1 in any channel]
/scores

[Bot response visible to everyone]
🏆 Top 10 Leaderboard

1. Player3
   15 points (8 wins)

2. Player1
   12 points (6 wins)

3. Player2
   8 points (5 wins)

4. Player4
   5 points (3 wins)

...
```

---

### Example 7: Checking Current Round

```
[Player1 in any channel]
/current

[Bot response - Round active but not locked]
Round 6
Status: 🟢 Active
Time Elapsed: 0:02:15

---

[Player1 uses /current after someone won]
/current

[Bot response - Round active and locked]
Round 6
Status: 🔒 Locked
Time Elapsed: 0:03:42
Winner: Player2 (+2 points)
```

---

## Implementation Files

Here's what each file will contain:

### 1. bot.py (~50 lines)
- Import discord.py and dotenv
- Load bot token from `.env`
- Create bot client with intents
- Load both cogs (admin, game)
- Sync slash commands on ready
- Error handling for missing token
- Start the bot

### 2. cogs/game.py (~200 lines)
- GameCog class
- Game state variables (active, artist, title, locked, etc.)
- on_message listener (processes guesses)
- /scores command (show leaderboard)
- /current command (show round status)
- Integration with answer_checker and data_manager

### 3. cogs/admin.py (~250 lines)
- AdminCog class
- /set_channel command
- /start_round command (with validation)
- /end_round command
- /set_answer command
- /show_answer command
- /reset_scores command
- All commands use permission check

### 4. utils/answer_checker.py (~40 lines)
- check_answer() function
- Whitespace normalization
- Uppercase detection
- Exact match logic
- Return (is_correct, points, match_type)

### 5. utils/data_manager.py (~150 lines)
- DataManager class
- load_config() and save_config()
- load_scores() and save_scores()
- add_score() (update user entry)
- get_leaderboard() (sorted by score, then name)
- Atomic write operations
- Error handling for corrupted JSON

### 6. utils/checks.py (~20 lines)
- has_manage_channels() permission check
- Returns True/False
- Sends ephemeral error if False

### 7. requirements.txt (~3 lines)
```
discord.py>=2.3.0
python-dotenv>=1.0.0
```

### 8. .env.example (~2 lines)
```
DISCORD_TOKEN=your_bot_token_here
```

### 9. .gitignore (~10 lines)
```
.env
venv/
__pycache__/
*.pyc
.vscode/
.idea/
```

### 10. data/config.json (~4 lines)
```json
{
  "game_channel_id": null,
  "admin_permission": "manage_channels"
}
```

### 11. data/scores.json (~1 line)
```json
{}
```

### 12. README.md (~300 lines)
Complete setup guide covering:
- What Blindy2 does
- Creating Discord bot (step-by-step with screenshots description)
- Installing Python dependencies
- Configuring .env
- Running the bot
- First-time setup (/set_channel)
- Commands reference
- How to play
- Troubleshooting common issues

---

## Setup Process

This is what you'll do AFTER the code is written.

### Phase 1: Create Discord Bot

1. **Go to Discord Developer Portal**
   - URL: https://discord.com/developers/applications
   - Log in with your Discord account

2. **Create Application**
   - Click "New Application"
   - Name: "Blindy2"
   - Click "Create"

3. **Create Bot User**
   - Left sidebar → Click "Bot"
   - Click "Add Bot" → Confirm
   - This creates the bot account

4. **Enable Privileged Intents** (CRITICAL!)
   - Under "Privileged Gateway Intents"
   - Enable: ✅ **MESSAGE CONTENT INTENT**
   - Enable: ✅ **SERVER MEMBERS INTENT** (optional but good)
   - Click "Save Changes"
   - **Without this, the bot cannot read messages!**

5. **Get Bot Token**
   - Under "Bot" section
   - Click "Reset Token" (or "Copy" if first time)
   - Copy the token (long string of letters/numbers)
   - **NEVER share this publicly!** It's like a password.

6. **Generate Invite Link**
   - Left sidebar → "OAuth2" → "URL Generator"
   - Select scopes:
     - ✅ `bot`
     - ✅ `applications.commands`
   - Select bot permissions:
     - ✅ Read Messages/View Channels
     - ✅ Send Messages
     - ✅ Embed Links
     - ✅ Read Message History
   - Copy the generated URL at the bottom

7. **Invite Bot to Your Server**
   - Paste the URL in your browser
   - Select your Discord server
   - Click "Authorize"
   - Complete captcha if prompted
   - Bot appears in your server (offline until you run the code)

---

### Phase 2: Install Python & Dependencies

1. **Check Python Version**
   ```bash
   python3 --version
   ```
   - Must be 3.9 or higher
   - If not installed, download from python.org

2. **Navigate to Project Folder**
   ```bash
   cd /home/salsoysa/Projects/blind_bot
   ```

3. **Create Virtual Environment**
   ```bash
   python3 -m venv venv
   ```
   - This creates a folder called `venv/` with isolated Python environment

4. **Activate Virtual Environment**
   
   **On Linux/Mac:**
   ```bash
   source venv/bin/activate
   ```
   
   **On Windows:**
   ```bash
   venv\Scripts\activate
   ```
   
   - Your terminal prompt should now show `(venv)` at the start

5. **Install Dependencies**
   ```bash
   pip install -r requirements.txt
   ```
   - This installs discord.py and python-dotenv

---

### Phase 3: Configure Bot

1. **Create .env File**
   ```bash
   cp .env.example .env
   ```
   - Or manually create a file named `.env`

2. **Add Your Bot Token**
   - Open `.env` in a text editor
   - Replace `your_bot_token_here` with the token you copied earlier
   - Example:
     ```
     DISCORD_TOKEN=MTIzNDU2Nzg5MDEyMzQ1Njc4OQ.GhJKlM.nOpQrStUvWxYz1234567890AbCdEfGhIjKlMn
     ```
   - Save and close

3. **Verify File Structure**
   ```
   blind_bot/
   ├── bot.py ✓
   ├── cogs/ ✓
   ├── utils/ ✓
   ├── data/
   │   ├── config.json ✓
   │   └── scores.json ✓
   ├── .env ✓ (with your token)
   ├── requirements.txt ✓
   └── venv/ ✓
   ```

---

### Phase 4: Run the Bot

1. **Start the Bot**
   ```bash
   python bot.py
   ```
   - You should see: `Blindy2 is online!`
   - Leave this terminal window open (bot runs here)

2. **Check Discord**
   - Go to your Discord server
   - Bot should now show as "Online" (green circle)

3. **Test Slash Commands**
   - Type `/` in any channel
   - You should see Blindy2's commands in the menu
   - If not, wait 1-2 minutes (Discord caches commands)

---

### Phase 5: First-Time Setup

1. **Set Game Channel**
   - Go to the channel where you want the game (e.g., `#blind-test`)
   - Type: `/set_channel`
   - Bot confirms: "✅ Game channel set to #blind-test"

2. **Test Your First Round**
   ```
   /start_round artist:queen title:bohemian rhapsody
   ```
   - Bot announces in game channel: "🎵 Round 1 started! Start guessing!"

3. **Test Guessing**
   - In the game channel, type: `queen`
   - Bot should respond: "✅ [Your Name] got the artist! (+1 point) 🔒 Question locked!"

4. **End the Round**
   ```
   /end_round
   ```
   - Bot shows summary with answer and winner

5. **Check Scores**
   ```
   /scores
   ```
   - Should show leaderboard with your name and 1 point

**Success!** Your bot is working. 🎉

---

### Phase 6: Stopping the Bot

**To stop the bot:**
1. Go to the terminal where bot.py is running
2. Press `Ctrl+C`
3. Bot goes offline in Discord

**To start again:**
1. Make sure virtual environment is activated (`source venv/bin/activate`)
2. Run `python bot.py`

---

## Troubleshooting

### Bot doesn't come online

**Possible causes:**
1. **Wrong token** → Check `.env` file, verify token from Developer Portal
2. **Intents not enabled** → Go to Developer Portal → Bot → Enable "Message Content Intent"
3. **Network issues** → Check internet connection
4. **Python errors** → Read error messages in terminal

### Bot doesn't respond to slash commands

**Possible causes:**
1. **Commands not synced** → Wait 1-2 minutes, or restart bot
2. **Bot offline** → Check terminal, restart bot.py
3. **Permissions** → Make sure bot has "Send Messages" permission in that channel
4. **Wrong server** → Make sure you invited bot to the right server

### Bot doesn't read guesses

**Possible causes:**
1. **Message Content Intent disabled** → Enable in Developer Portal, restart bot
2. **Wrong channel** → Use `/set_channel` in the game channel first
3. **Round not started** → Use `/start_round` first
4. **Round locked** → Someone already answered, use `/end_round` and start new round

### "Missing Permissions" error when using admin commands

**Possible causes:**
1. **You don't have "Manage Channels" permission** → Ask server owner to give you a role with this permission
2. **You're not server owner** → Server owner always has all permissions

### Scores not saving

**Possible causes:**
1. **File permissions** → Make sure `data/scores.json` is writable
2. **Corrupted JSON** → Delete `scores.json` (or backup first), bot will create new one
3. **Disk full** → Check available disk space

---

## Next Steps After Reading This

Once you've read and understood this plan:

1. **Ask questions** about anything unclear
2. **Confirm you're ready** for implementation
3. **I'll create all the files** with complete, working code
4. **You'll follow the setup process** to get your bot running

Take your time reviewing this. Understanding the architecture now will help you:
- Modify the bot later (add features, fix bugs)
- Troubleshoot issues when they arise
- Explain to others how it works

Let me know when you're ready to proceed! 🚀
