# 🎮 PlayBound

A high-performance, multi-server Discord bot designed for community engagement through automated games, persistent economies, and interactive events.

## 🚀 Core Features

- **Multi-Server Ready:** Complete data isolation between servers. Points, settings, and games in your test server never interfere with your live server.
- **Persistent Economy & Shop:** A robust points system where users earn currency for winning games, participating in events, and maintain a **Daily Streak** for bonuses. Points can be spent on items in the `/shop`.
- **Global Factions (New!):** Users can join global teams (Dragons 🐉, Wolves 🐺, Eagles 🦅) and contribute their points to a cross-server global leaderboard.
- **Autopilot System [💎 Premium]:** Schedule games like Trivia and Serverdle to run automatically on a recurring timer without admin intervention.
- **Dynamic Game Suites:** Host various game types including Trivia, Guess The Number, Wordle clones (with visual board generation!), and Music Trivia.
- **One-Word Story Mode:** Create a dedicated channel where users collaboratively build stories, one word at a time.
- **Crash Recovery:** Built-in system that automatically detects and handles orphaned game threads if the bot experiences an unexpected restart.
- **Admin Tools:** Extensive configuration for announcements, welcome messages, role rewards, and point adjustments.

## 🎮 Available Games

### 🧠 Trivia Match (`/trivia`)
A live multiplayer match. The first person to click the correct answer button wins the round.
- Supports multiple categories and difficulty levels.
- Configurable question counts and point values.
- **[💎 Premium]** Can be set to repeat automatically every X hours (`repeat_hrs`).

### 🏃 Trivia Sprint (`/triviasprint`)
A speedrun challenge! Users click "Start" and get their own private set of 15 questions.
- Individual timers track speed.
- Final leaderboard ranks users by correct answers, then by time.

### ⚔️ 1v1 Trivia Duels (`/duel`)
Players can challenge each other to a high-stakes rapid-fire trivia match.
- Users wager their own points. Winner takes the entire pot!

### 🎲 Dice Roll Tournament (`/tournament`)
Start a multi-round bracketed tournament where users "roll off" to advance.
- Fully automated: The bot handles registration, bracket generation, and match simulation.
- **Winner Takes All:** The last player standing wins the entire pot (Initial Pot + Entry Fees).
- Includes support for "Byes" and custom entry fees.

### 🔢 Guess The Number (`/guessthenumber`)
A classic guessing game where users try to guess a secret number within a specified range.
- The bot tracks all guesses and prevents duplicates.
- Automatically finds the winner (closest guess) when the timer expires.

### 🟩 Serverdle (`/startserverdle`)
A community Wordle replica! Users attempt to guess a 5-letter word within 6 tries.
- Shared secret word for the server.
- **Visual Board Generation:** High-quality generated images for every guess.
- **[💎 Premium]** Can be set to repeat automatically every X hours (`repeat_hrs`).

### 🎵 Name That Tune (`/namethattune`)
Join a voice channel and guess the song titles as the bot plays snippets from your local music library.
- Uses fuzzy matching to allow for minor typos.
- Tracks wins and guess speed across multiple rounds.

### 📝 Unscramble (`/unscramble`)
A fast-paced word game where users must unscramble phrases based on a clue.
- Keeps spaces in mind while shuffling letters.
- Multiple rounds with point rewards for the fastest solver.

### 🖼️ Caption Contest (`/caption`)
The bot pulls a random image (Cat, Dog, Fox, etc.) and users compete to write the funniest caption.
- Winner is decided by community reactions (votes) on the entries.

### 📝 One-Word Story (`/set_story_channel`)
Collaborate on a story one word at a time. The bot enforces strict rules (no double-posting, exactly one word) and handles archiving.

### 🎁 Giveaways (`/giveaway`)
Host automated giveaways with configurable winners and entry cooldowns.
- Includes a "Cooldown" feature to prevent the same user from winning too frequently.

## 🛍️ Economy, Shop & Customization

Users can check their points with `/profile` and view available items with `/shop`.

### Player Economy
- `/daily`: Claim free points every 24 hours. **[💎 Premium]** users receive bonus points!
- `/pay`: Safely transfer points to other users to create a player-driven economy.
- **Daily Streaks:** Earn a bonus multiplier for every consecutive day you participate in a game.

### The Shop & Inventory
- **Consumables:** Buy items like "Trivia Hints" or "Streak Shields" to use in games.
- **Cosmetics:** Buy and `/equip` custom name colors and special profile badges.
- **Inventory:** View your items anytime with `/inventory`.
- **[💎 Premium] Exclusives:** Special badges (Diamond Badge) and colors (Crystal Blue) are reserved for subscribers.

### Server Pro Shops [💎 Premium]
Server owners with Premium can create their own custom economies:
- `/server_shop_add`: Create custom roles, badges, or name colors that users can buy with points specifically in your server.
- `/server_shop_remove`: Remove custom items from your server's shop.

## 🌐 Global Factions

Connect your server to the global PlayBound meta!
- `/faction join`: Users can pledge allegiance to one of three global factions: Dragons 🐉, Wolves 🐺, or Eagles 🦅. Every point they earn in any server contributes to their faction's global total.
- `/faction`: View your personal contribution and faction stats.
- `/factions`: View the live Global Faction Leaderboard to see which team is currently winning the war.

## 🛠️ Admin Commands

### Configuration
- `/set_announcement_channel`: Designate where game starts and winners are announced.
- `/set_welcome_channel`: Set the channel for member greetings.
- `/add_welcome_message`: Add a message to the rotation pool. Use `{user}` to mention the new member.
- `/list_welcome_messages`: View all greeting variations in rotation.
- `/remove_welcome_message`: Remove a message from the pool.
- `/set_birthday_channel`: Set where the bot should shout out birthdays.
- `/add_birthday_message`: Add a birthday message to rotation.
- `/set_achievement_channel`: Set where trophy unlocks are announced.
- `/set_leaderboard_channel`: Set a dedicated channel for a live-updating Top 10 board.
- `/set_story_channel`: Designate a channel for the One-Word Story game.
- `/set_role_reward`: Link an achievement key to a Discord Role ID for automatic granting.
- `/set_manager_role`: Delegate game-starting permissions to a specific role.

### Moderation & Economy
- `/adjustpoints`: Manually give or take points from a user with an optional reason.
- `/wipe_leaderboard`: Reset all server points to zero (useful for new seasons).
- `/endgame`: Forcefully terminate any active game in the current thread.
- `/add_redirect`: Set up auto-replies to prompt users to move specific topics to another location.
  - **Keywords:** Triggers on single words or comma-separated lists (e.g., `help,support`).
  - **Targets:** Link to a specific `#channel`, a direct message URL, or an external link.
  - **Custom Messages:** Provide an optional custom reply to explain the redirect to the user.
- `/remove_redirect`: Remove an existing auto-reply trigger.

## 🛠️ Support Commands
- `/support`: Get an invite link to the official PlayBound support server.
- `/ticket`: Open a private support ticket to report a bug, suggest an idea, or get help from the developers.

## 🏆 Achievements
The bot includes a built-in trophy system for milestones like:
- **🥇 First Class:** Win your first game or giveaway.
- **⚡ Speed Demon:** Complete a Trivia Sprint in under 60 seconds.
- **🧠 Wordle Wizard:** Solve a Serverdle in 3 or fewer guesses.
- **💬 Chatterbox:** Reach 100 messages sent in the server.
- **⭐ Loyal Citizen:** Reach 50 total points.
- **🤴 Trivia King / Serverdle Master / Guess Master:** Earn 5 wins in a specific game mode.
