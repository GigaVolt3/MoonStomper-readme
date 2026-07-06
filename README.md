# MoonStomper Discord Bot — Technical Documentation

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Directory Structure](#3-directory-structure)
4. [Environment Variables](#4-environment-variables)
5. [Dependencies](#5-dependencies)
6. [Startup & Deployment](#6-startup--deployment)
7. [Database Architecture](#7-database-architecture)
8. [Discord Commands](#8-discord-commands)
9. [Event Handlers](#9-event-handlers)
10. [Interaction Handlers](#10-interaction-handlers)
11. [AI/LLM System](#11-aillm-system)
12. [RPG Game Engine](#12-rpg-game-engine)
13. [Web Dashboard](#13-web-dashboard)
14. [Security Architecture](#14-security-architecture)
15. [Failover & Sync](#15-failover--sync)

---

## 1. Project Overview

**MoonStomper** is a Discord bot built for the game *Where Winds Meet* (Wuxia MMORPG). It is a dual-process Node.js application (~32,000+ lines across ~90 files) combining:

- **Wanderer Nameplate/Profile system** for guild members
- **Full RPG game engine** (turn-based combat, raids, story, crafting, exploration)
- **AI chatbot** powered by Groq's LLM API with 8-model cascade
- **Web dashboard** with wiki, event management, and admin tools
- **Turso/libsql** database backend with cloud sync

| Property | Value |
|----------|-------|
| Runtime | Node.js >= 22 |
| Module System | ES Modules (`"type": "module"`) |
| Primary Entry | `src/index.js` (bot), `src/web.js` (dashboard) |
| Dashboard URL | `https://moonstomper.wisp.uno` |
| Dashboard Port | `9303` |

---

## 2. Architecture

```
+-------------------------------------------------------------+
|                     PM2 Process Manager                      |
|  +------------------+       +------------------+             |
|  |    moon-bot       |       |    moon-web       |            |
|  |  (src/bot.js)     |       |  (src/web.js)     |            |
|  |                   |       |                   |            |
|  |  +-------------+  |       |  +-------------+  |            |
|  |  | Discord.js   |  |       |  | Express v5  |  |            |
|  |  | Client       |  |       |  | Server      |  |            |
|  |  +------+------+  |       |  +------+------+  |            |
|  |         |         |       |         |         |            |
|  |  +------+------+  |       |  +------+------+  |            |
|  |  | Command     |  |       |  | REST API    |  |            |
|  |  | Router      |  |       |  | + WebSocket |  |            |
|  |  +------+------+  |       |  +------+------+  |            |
|  |         |         |       |         |         |            |
|  |  +------+------+  |       |  +------+------+  |            |
|  |  | AI/LLM      |  |       |  | botBridge   |  |            |
|  |  | Pipeline    |  |       |  | eventBus    |  |            |
|  |  +------+------+  |       |  +------+------+  |            |
|  +--------+---------+       +--------+---------+             |
|           |                          |                        |
|  +--------+--------------------------+--------+              |
|  |         Shared SQLite Databases              |              |
|  |  memory.db          game.db                  |              |
|  |  (libsql + Turso embedded replica)          |              |
|  +----------------------------------------------+              |
+-------------------------------------------------------------+
```

### Process Communication
- Both processes share SQLite databases via `libsql`
- Bot <-> Web communication via `botBridge.js` (control) and `eventBus.js` (pub/sub)
- WebSocket for live dashboard updates

---

## 3. Directory Structure

```
MoonStomper discord bot/
|-- .env                          # Environment variables (secrets)
|-- .gitignore
|-- package.json
|-- ecosystem.config.cjs          # PM2 config (2 processes)
|-- run_bot.bat                   # Windows batch launcher
|-- ARCHITECTURE_MAP.md
|
|-- src/
|   |-- index.js                  # Main bot entry (phased startup)
|   |-- bot.js                    # Standalone bot entry
|   |-- web.js                    # Web process entry (1s delay)
|   |-- web-core.js               # Web process init (dummy client)
|   |-- register-commands.js      # Slash command registration
|   |
|   |-- commands/                 # Meta/Admin Commands (16 files)
|   |   |-- profile.js            # /profile -- Wanderer Nameplate
|   |   |-- admin.js              # /admin -- Leader panel (591 lines)
|   |   |-- help.js               # /help -- Context-aware help
|   |   |-- vacation.js           # /vacation -- Leave of Absence
|   |   |-- lookup.js             # /lookup -- Member search
|   |   |-- stats.js              # /stats -- Guild statistics
|   |   |-- invite.js             # /invite vc -- VC invite
|   |   |-- key.js                # /key -- API key management
|   |   |-- status.js             # /status -- AI health
|   |   |-- register.js           # /register -- Server registration
|   |   |-- server.js             # /server -- Owner server mgmt
|   |   |-- playercard.js         # /playercard -- IGN lookup
|   |   |-- wiki.js               # /wiki -- Wiki link
|   |   |-- mystats.js            # /mystats -- Personal stats
|   |   |-- memories.js           # /memories -- AI memory view
|   |   |-- event.js              # /event -- Event wizard
|   |
|   |-- game/
|   |   |-- commands/             # Game Commands (19 files)
|   |   |   |-- start.js          # Character creation
|   |   |   |-- adventure.js      # Wilderness combat
|   |   |   |-- profile.js        # RPG profile
|   |   |   |-- inventory.js      # Inventory UI
|   |   |   |-- equip.js          # Equip items
|   |   |   |-- upgrade.js        # Train attributes
|   |   |   |-- skills.js         # Martial skills
|   |   |   |-- breakthrough.js   # Realm breakthrough
|   |   |   |-- hunt.js           # Elite hunting
|   |   |   |-- explore.js        # Region exploration
|   |   |   |-- map.js            # World map
|   |   |   |-- shop.js           # Merchant
|   |   |   |-- craft.js          # Crafting
|   |   |   |-- sect.js           # Faction join
|   |   |   |-- boss.js           # World boss raid
|   |   |   |-- raid.js           # Co-op raids
|   |   |   |-- arena.js          # PvP duel
|   |   |   |-- story.js          # Story campaign
|   |   |   |-- leaderboard.js    # Rankings
|   |   |
|   |   |-- core/
|   |   |   |-- manager.js        # Game router (636 lines)
|   |   |   |-- stateManager.js   # Player state machine (592 lines)
|   |   |   |-- skills.js         # Skill definitions
|   |   |   |-- sects.js          # 12 faction definitions
|   |   |
|   |   |-- combat/
|   |   |   |-- combat.js         # Core combat engine (909 lines)
|   |   |   |-- combatNarrator.js # LLM combat narration
|   |   |
|   |   |-- exploration/exploration.js
|   |   |-- inventory/loot.js     # Loot generation (251 lines)
|   |   |-- raid/
|   |   |   |-- raidEngine.js     # Raid logic (505 lines)
|   |   |   |-- raidVisuals.js    # Raid UI
|   |   |-- story/
|   |   |   |-- story.js          # Story rendering
|   |   |   |-- storyEngine.js    # LLM story progression (955 lines)
|   |   |-- database/db.js        # Game DB layer (759 lines)
|   |   |-- assets/art.js         # Weapon/monster art
|   |
|   |-- database/
|   |   |-- db.js                 # Profile/guild DB (899 lines)
|   |   |-- memoryDb.js           # Memory/personality/wiki DB (2002 lines)
|   |   |-- eventsDb.js           # Events DB (140 lines)
|   |   |-- guideSchemas.js       # Wiki schemas (275 lines)
|   |   |-- personalityEngine.js  # Communication style engine
|   |   |-- turso.js              # Turso sync (151 lines)
|   |
|   |-- events/
|   |   |-- messageCreate.js      # AI chat handler (782 lines)
|   |   |-- interactionCreate.js  # Central interaction router
|   |   |-- guildEvents.js        # Guild lifecycle (295 lines)
|   |   |-- dashboardEvents.js    # Dashboard subscriptions
|   |
|   |-- interactions/
|   |   |-- buttons.js            # Button handler (1641 lines)
|   |   |-- selects.js            # Select menu handler (458 lines)
|   |   |-- modals.js             # Modal handler (298 lines)
|   |   |-- eventsInteraction.js  # Event interactions (315 lines)
|   |
|   |-- services/
|   |   |-- eventBus.js           # Internal pub/sub
|   |   |-- botBridge.js          # Dashboard <-> Bot control
|   |   |-- profileService.js     # Profile CRUD
|   |   |-- vacationService.js    # Vacation management
|   |   |-- keyService.js         # API key management
|   |   |-- guildService.js       # Guild settings
|   |   |-- inviteService.js      # Invitations
|   |
|   |-- utils/                    # 27 utility files
|   |   |-- groqChat.js           # AI orchestration (1685 lines)
|   |   |-- modelRouter.js        # 8-model cascade (652 lines)
|   |   |-- messageClassifier.js  # Message classification (356 lines)
|   |   |-- modelPipeline.js      # 3-model pipeline
|   |   |-- mediaHandlers.js      # Image/audio handlers
|   |   |-- criticPasses.js       # Multi-model debate
|   |   |-- cloudflareAI.js       # Cloudflare Workers AI
|   |   |-- resourceManager.js    # CPU/RAM monitoring (418 lines)
|   |   |-- syncManager.js        # Primary/secondary failover
|   |   |-- backgroundThinking.js # Background AI processing
|   |   |-- botDreams.js          # Nightly consolidation
|   |   |-- maintenance.js        # Daily scheduler
|   |   |-- nameplate.js          # Profile card builder
|   |   |-- innerWays.js          # Martial arts defs (396 lines)
|   |   |-- crypto.js             # AES-256-GCM encryption
|   |   |-- dbLock.js             # SQLite WAL lock
|   |   |-- commandQueue.js       # Command queuing
|   |
|   |-- dashboard/
|   |   |-- server.js             # Express server (4476 lines)
|   |   |-- public/               # SPA frontend
|   |
|   |-- workers/
|       |-- imageWorker.js        # SVG->PNG via sharp
|       |-- imageWorkerPool.js    # Worker pool
```

---

## 4. Environment Variables

| Variable | Purpose | Security |
|----------|---------|----------|
| `BOT_ROLE` | Instance role: `SECONDARY` or `PRIMARY` | -- |
| `DEV_MODE` | Development mode toggle | -- |
| `THREE_MODEL_PIPELINE` | Enable 3-model pipeline | -- |
| `DISCORD_CLIENT_ID` | Discord application ID | -- |
| `DISCORD_CLIENT_SECRET` | Discord OAuth2 secret | **CRITICAL** |
| `DISCORD_TOKEN` | Bot authentication token | **CRITICAL** |
| `DISCORD_TEST_GUILD_ID` | Test guild ID | -- |
| `GROQ_API_KEY` | Primary Groq API key | **CRITICAL** |
| `ENCRYPTION_SECRET` | AES-256-GCM key for API key encryption | **CRITICAL** |
| `GROQ_MODEL` | Primary text model: `llama-3.1-8b-instant` | -- |
| `GROQ_MODEL_F1`-`F7` | 7 fallback models | -- |
| `GROQ_VISION_MODEL` | Vision: `meta-llama/llama-4-scout-17b-16e-instruct` | -- |
| `GROQ_AUDIO_MODEL` | STT: `whisper-large-v3-turbo` | -- |
| `GROQ_AUDIO_FALLBACK` | STT fallback: `whisper-large-v3` | -- |
| `SYNC_ENCRYPTION_KEY` | AES-256-GCM key for sync payloads | **CRITICAL** |
| `TURSO_MEMORY_DATABASE_URL` | Turso memory DB URL | -- |
| `TURSO_MEMORY_AUTH_TOKEN` | Turso memory auth token | **CRITICAL** |
| `TURSO_GAME_DATABASE_URL` | Turso game DB URL | -- |
| `TURSO_GAME_AUTH_TOKEN` | Turso game auth token | **CRITICAL** |
| `DASHBOARD_PORT` | Web dashboard port: `9303` | -- |
| `DASHBOARD_URL` | Public dashboard URL | -- |
| `JWT_SECRET` | JWT signing secret | **CRITICAL** |
| `CLOUDFLARE_API_TOKEN` | Cloudflare Workers AI token | **CRITICAL** |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare account ID | -- |
| `BOT_OWNER_ID` | Bot owner Discord ID | -- |

---

## 5. Dependencies

### Production

| Package | Version | Purpose |
|---------|---------|---------|
| `discord.js` | ^14.26.4 | Discord API framework |
| `@discordjs/voice` | ^0.19.2 | Voice channel support |
| `@libsql/client` | ^0.17.4 | Turso embedded replica |
| `libsql` | ^0.5.29 | SQLite-compatible DB |
| `express` | ^5.2.1 | Web server |
| `express-rate-limit` | ^8.5.2 | API rate limiting |
| `compression` | ^1.8.1 | HTTP compression |
| `cookie-parser` | ^1.4.7 | Cookie parsing |
| `jsonwebtoken` | ^9.0.3 | JWT auth |
| `ws` | ^8.21.0 | WebSocket |
| `multer` | ^2.2.0 | File uploads |
| `sharp` | ^0.33.4 | SVG->PNG processing |
| `dotenv` | ^16.4.5 | Env loading |
| `lru-cache` | ^11.5.1 | In-memory caching |
| `pm2` | ^7.0.1 | Process manager |
| `simple-git` | ^3.36.0 | Git auto-update |

### Dev

| Package | Version | Purpose |
|---------|---------|---------|
| `eslint` | ^9.0.0 | Linting |
| `@eslint/js` | ^9.0.0 | ESLint config |

---

## 6. Startup & Deployment

### PM2 Configuration (`ecosystem.config.cjs`)

```javascript
module.exports = {
  apps: [
    {
      name: 'moon-bot',
      script: 'src/bot.js',
      max_memory_restart: '350M',
      restart_delay: 3000,
      node_args: '--max-old-space-size=256'
    },
    {
      name: 'moon-web',
      script: 'src/web.js',
      max_memory_restart: '250M',
      restart_delay: 3000,
      node_args: '--max-old-space-size=256'
    }
  ]
};
```

### Phased Startup Sequence

```
Phase 1 (Core)        -> JWT validation, memory DB init, cache registration
Phase 1b (GameDB)     -> Game database init (parallel with Core)
Phase 2 (Services)    -> Event handler imports, Turso sync
Phase 3 (Client)      -> Discord client creation, command registration
Phase 4 (Login)       -> Event handlers, login, sync manager start
Phase 5 (GuildSync)   -> Lightweight guild approval registration
Phase 6 (CacheWarm)   -> Send pending welcome/IGN DMs
Phase 7 (Background)  -> Heartbeat intervals, daily Turso sync
```

### Graceful Shutdown

Handles: `SIGINT`, `SIGTERM`, `SIGHUP`, `SIGBREAK`

1. Stops all schedulers, clears intervals
2. Executes Turso backup
3. Sets status channels offline
4. Destroys Discord client, closes databases

---

## 7. Database Architecture

### Database 1: `memory.db` (Profile + Memory + Wiki)

| Table | Purpose |
|-------|---------|
| `profiles` | User profiles (JSON: loadout, inner_ways, sect, vacation, relationships) |
| `guild_approvals` | Server approval status (pending/approved/blocked) |
| `guild_configs` | Per-guild settings (leave_channel_id, game_enabled) |
| `system_stats` | Bot state (maintenance_mode, turso_db_mode) |
| `user_memories` | AI memory facts per user |
| `user_personality` | Per-user communication style |
| `chat_history` | Conversation history (rolling buffer) |
| `image_descriptions` | Cached image analysis |
| `command_history` | Command execution metrics |
| `active_member_threads` | Active member thread mappings |
| `contributed_keys` | Community API keys (encrypted) |
| `key_token_usage` | Per-key token usage |
| `guide_pages` | Wiki page content |
| `guide_changes` | Wiki edit proposals |
| `guide_versions` | Wiki version history |
| `guide_contributors` | Wiki contributor stats |
| `wiki_categories` | Wiki categories |
| `wiki_comments` | Wiki comments |
| `wiki_page_locks` | Wiki editing locks |
| `wiki_user_roles` | Wiki permission roles |
| `wiki_media` | Uploaded media library |
| `events` | Guild events |
| `event_participants` | Event signups |
| `event_templates` | Saved event templates |

### Database 2: `game.db` (RPG Game State)

| Table | Purpose |
|-------|---------|
| `players` | Character stats, realm, HP, attributes, gold, stamina |
| `player_items` | Inventory items with stats/rarity/tuning |
| `player_skills` | Learned/upgraded martial skills |
| `player_story` | Story campaign progress |
| `story_nodes` | Story narrative nodes |
| `combat_states` | Active combat state |
| `cooldowns` | Command cooldown timers |
| `raid_state` | Active raid instance |

### Caching Strategy

- **Profile reads:** LRU cache (500 entries, 10 min TTL)
- **Turso reads:** LRU cache (500 entries, 5 min TTL)
- **Memory pressure:** Caches cleared on high/emergency CPU/RAM zones
- **WAL contention:** Mitigated by 1s web delay + `dbLock.js` promise-based lock

---

## 8. Discord Commands

### Meta/Admin Commands (16)

| Command | File | Description |
|---------|------|-------------|
| `/profile` | `commands/profile.js` | View/edit Wanderer Nameplate |
| `/admin` | `commands/admin.js` | Leader panel (settings, sync, toggles) |
| `/help` | `commands/help.js` | Context-aware help embed |
| `/vacation` | `commands/vacation.js` | File Leave of Absence |
| `/lookup` | `commands/lookup.js` | Search members by filters |
| `/stats` | `commands/stats.js` | Guild-wide statistics |
| `/invite vc` | `commands/invite.js` | DM active players to join VC |
| `/key` | `commands/key.js` | API key contribution panel |
| `/status` | `commands/status.js` | AI network health display |
| `/register` | `commands/register.js` | Server registration request |
| `/server` | `commands/server.js` | Bot owner server management |
| `/playercard` | `commands/playercard.js` | IGN-based player lookup |
| `/wiki` | `commands/wiki.js` | Wiki link embed |
| `/mystats` | `commands/mystats.js` | Personal usage stats |
| `/memories` | `commands/memories.js` | View AI memory about you |
| `/event` | `commands/event.js` | Event creation wizard |

### RPG Game Commands (19)

| Command | File | Description |
|---------|------|-------------|
| `/start` | `game/commands/start.js` | Character creation (6 archetypes) |
| `/rpg-profile` | `game/commands/profile.js` | Character stats, Town Square |
| `/adventure` | `game/commands/adventure.js` | Wilderness combat |
| `/hunt` | `game/commands/hunt.js` | Elite monster hunting |
| `/explore` | `game/commands/explore.js` | Region exploration |
| `/map` | `game/commands/map.js` | World/region map |
| `/inventory` | `game/commands/inventory.js` | Paginated inventory |
| `/equip` | `game/commands/equip.js` | Equip weapon/armor/accessories |
| `/upgrade` | `game/commands/upgrade.js` | Train attributes |
| `/skills` | `game/commands/skills.js` | Martial skill management |
| `/breakthrough` | `game/commands/breakthrough.js` | Realm breakthrough boss |
| `/sect` | `game/commands/sect.js` | Join faction (12 sects) |
| `/shop` | `game/commands/shop.js` | Town merchant |
| `/craft` | `game/commands/craft.js` | Item crafting |
| `/boss` | `game/commands/boss.js` | World boss raid |
| `/raid` | `game/commands/raid.js` | Co-op party raids |
| `/arena` | `game/commands/arena.js` | PvP duel |
| `/story` | `game/commands/story.js` | Solo story campaign |
| `/leaderboard` | `game/commands/leaderboard.js` | Global rankings |

---

## 9. Event Handlers

### Discord Events

| Event | File | Description |
|-------|------|-------------|
| `messageCreate` | `events/messageCreate.js` | AI chat handler -- processes messages through LLM pipeline |
| `interactionCreate` | `events/interactionCreate.js` | Central router for all interactions |
| `guildCreate` | `events/guildEvents.js` | Auto-registers new guilds |
| `guildMemberAdd` | `events/guildEvents.js` | Welcome nameplate DM |
| `guildMemberRemove` | `events/guildEvents.js` | Cleanup relationships/threads |
| `guildMemberUpdate` | `events/guildEvents.js` | Sync name/role changes |
| `dashboardEvents` | `events/dashboardEvents.js` | Dashboard settings updates |

### Message Processing Pipeline

```
User Message
    |
    v
Dedup Check (recent messages)
    |
    v
Guild Approval Check
    |
    v
Message Classification (simple/complex/extreme)
    |
    v
LLM Chain (model cascade routing)
    |
    v
Memory Extraction (background)
    |
    v
Personality Update (background)
    |
    v
Reply to User
```

---

## 10. Interaction Handlers

### Buttons (`interactions/buttons.js` -- 1641 lines)

| Prefix | Purpose |
|--------|---------|
| `btn_dm_*` | DM verification (partner/disciple requests) |
| `btn_vacation_*` | Vacation management (submit, return, extend) |
| `btn_profile_*` | Profile customization (IGN, loadout, inner ways) |
| `btn_key_*` | API key contribution |
| `btn_admin_*` | Admin operations |
| `btn_owner_*` | Bot owner operations |
| `btn_game_*` | Game interactions (combat, exploration, raids) |
| `btn_help_*` | Help system |
| `btn_return_leave_*` | Return from leave |
| `btn_edit_game_username_*` | IGN editing |

### Select Menus (`interactions/selects.js` -- 458 lines)

| Prefix | Purpose |
|--------|---------|
| `select_vacation_duration_*` | Vacation duration |
| `select_weapons_*` | Weapon loadout |
| `select_sect_*` | Faction selection |
| `select_inner_way_*` | Skill slot assignment |
| `select_relationship_*` | Partner/disciple |
| `select_lookup_result_*` | Member lookup result |
| `select_owner_manage_server` | Server management |
| `event_wizard_*` | Event creation |
| `event_signup_*` | Event participation |

### Modals (`interactions/modals.js` -- 298 lines)

| Prefix | Purpose |
|--------|---------|
| `modal_game_username_*` | IGN entry |
| `modal_profile_bio_*` | Bio editing |
| `modal_vacation_*` | Vacation reason |
| `modal_leave_*` | Leave reason |
| `modal_*_extend_*` | Vacation extension |

---

## 11. AI/LLM System

### 8-Model Groq Cascade

```
Primary:  llama-3.1-8b-instant (fastest, highest daily quota)
    | rate limit/error
Fallback 1: qwen/qwen3-32b (highest RPM)
    |
Fallback 2: llama-3.3-70b-versatile (best reasoning)
    |
Fallback 3: openai/gpt-oss-120b
    |
Fallback 4: openai/gpt-oss-20b
    |
Fallback 5: groq/compound-mini (compound routing)
    |
Fallback 6: groq/compound
    |
Fallback 7: allam-2-7b (last resort)
```

### Specialized Models

| Task | Model |
|------|-------|
| Vision | `meta-llama/llama-4-scout-17b-16e-instruct` |
| Audio STT | `whisper-large-v3-turbo` (fallback: `whisper-large-v3`) |
| Memory Extraction | `allam-2-7b` |
| Image Generation | Cloudflare Workers AI (`@cf/stabilityai/stable-diffusion-xl-base-1.0`) |

### Key AI Features

- **Message Classification:** Routes to appropriate model based on complexity
- **Community Key Pool:** Users contribute Groq API keys (AES-256-GCM encrypted)
- **Key Rotation:** Automatic cycling on rate limit
- **Background Thinking:** Queue-based memory extraction
- **Bot Dreams:** Nightly memory consolidation and deduplication
- **3-Model Pipeline:** Optional architect/critic/finalizer multi-pass
- **Critic Passes:** Multi-model debate mode
- **Combat Narration:** LLM-generated battle descriptions
- **Story Engine:** LLM-driven dynamic narrative

---

## 12. RPG Game Engine

### Combat System (`game/combat/combat.js` -- 909 lines)

**Actions:** Attack, Heavy Attack, Defend, Skill, Item, Flee

**Stats Calculation:**
```
Final Stats = Base Attributes + Equipped Items + Sect Passive + Skills
```

**Enemy Scaling:** Player level +/- range (player-2 to player+3)

**Enemy Types:** 7 standard + 8 boss types

**Cooldown:** 6 seconds between adventures

### Character Progression

| System | Details |
|--------|---------|
| **Weapon Archetypes** | Sword, Spear, Fan, Umbrella, Twin Blades, Rope Dart |
| **Realms** | Beginner (15) -> Martial Disciple (20) -> Expert (30) -> Master (40) -> Grandmaster (50) -> Legend (60) -> Immortal (70) |
| **Attributes** | Power, Body, Defense, Agility, Momentum |
| **Sects** | 12 factions with unique passives and shop items |
| **Skills** | 6 Mystic Skills (Tai Chi, Meridian Touch, Celestial Seize, Cloud Steps, Golden Body, Glow of Fireflies) |

### Loot System (`game/inventory/loot.js` -- 251 lines)

| Rarity | Drop Rate |
|--------|-----------|
| Common | High |
| Uncommon | Medium |
| Rare | Low |
| Epic | Very Low |
| Legendary | Extremely Low |
| Mythic | Rare |
| Divine | Ultra Rare |

**Item Types:** Weapons (6), Armor (head/chest/arms/legs), Accessories (pendant/disc/ring)

**Tuning System:** 5 stat roll slots per item

### Raid System (`game/raid/raidEngine.js` -- 505 lines)

- **Bosses:** Dalang (Demon Steed), Wandering Swordsman, Corrupted General
- **Lobby Size:** 5 or 10 players
- **Shared Loot:** Distribution system

### Story Engine (`game/story/storyEngine.js` -- 955 lines)

- LLM-generated dynamic narrative
- Predefined root nodes with LLM-generated choices
- Background pre-generation of choice branches
- Cloudflare AI image generation per node
- Jaro-Winkler similarity for choice matching

### Regions

| Region | Base Level | Stamina Cost |
|--------|------------|--------------|
| Qinghe Forest | 1 | 10 |
| Kaifeng Outskirts | 25 | 15 |

---

## 13. Web Dashboard

**Server:** Express v5 (`src/dashboard/server.js` -- 4476 lines)

### Features

- **SPA Frontend:** `src/dashboard/public/`
- **Wiki System:** Pages, categories, version history, comments, roles, media
- **Event Management:** Create/manage events with templates
- **Admin Panel:** Guild approvals, module toggles, DB mode
- **Live Console:** WebSocket-streamed PM2 logs
- **Owner Panel:** Server management, restart, command reload
- **File Uploads:** Multer-based handling
- **API Rate Limiting:** Express-rate-limit middleware

### API Routes (80+)

| Category | Routes |
|----------|--------|
| Auth | `/api/auth/discord`, `/api/auth/callback`, `/api/auth/me` |
| Profile | `/api/profile`, `/api/profiles` |
| Guild | `/api/guilds`, `/api/guild/:id/config` |
| Admin | `/api/admin/*` |
| Owner | `/api/owner/*` |
| Wiki | `/api/wiki/*` |
| Events | `/api/events/*` |
| Game | `/api/game/*` |
| Keys | `/api/keys/*` |
| System | `/api/system/*` |

### Authentication Flow

```
Discord OAuth2 -> JWT Token (7-day expiry) -> httpOnly secure cookie
```

---

## 14. Security Architecture

### Authentication Layers

1. **Discord OAuth2** -> JWT (7-day expiry, httpOnly secure cookie)
2. **Role-Based Access:** `owner` -> `server_admin` -> `user`
3. **Wiki Roles:** `viewer` -> `contributor` -> `trusted_editor` -> `moderator` -> `admin`
4. **Live Discord Cache** for admin checks (5-minute LRU)

### Encryption

- **AES-256-GCM** for contributed API keys at rest
- **AES-256-GCM** for sync payload encryption
- **Storage Format:** `<iv_hex>:<authTag_hex>:<ciphertext_hex>`

### Guild Approval System

```
New Guild -> "pending" -> Bot Owner Approves -> "approved"
                              |
                              +-> Denies -> "blocked"
```

Per-guild module toggles: game, nameplate, AI

---

## 15. Failover & Sync

### Primary/Secondary Failover (`utils/syncManager.js` -- 333 lines)

- Both instances share the same bot token
- Heartbeat channel coordinates failover
- Only ACTIVE instance processes commands
- On activation: auto-register commands, start schedulers, initial guild sweeps

### Turso Cloud Sync (`database/turso.js` -- 151 lines)

- Daily automatic sync (24-hour interval)
- Monthly write limit tracking (10M writes cap)
- Embedded replica with local SQLite fallback
- Stale replica detection with file rotation

---

## Key File Paths

| Category | Key Files |
|----------|-----------|
| Entry Points | `src/index.js`, `src/bot.js`, `src/web.js`, `src/web-core.js` |
| Config | `.env`, `package.json`, `ecosystem.config.cjs` |
| AI Core | `src/utils/groqChat.js`, `src/utils/modelRouter.js`, `src/utils/messageClassifier.js` |
| Database | `src/database/db.js`, `src/database/memoryDb.js`, `src/game/database/db.js` |
| Game Engine | `src/game/combat/combat.js`, `src/game/core/stateManager.js`, `src/game/core/manager.js` |
| Dashboard | `src/dashboard/server.js` |
| Services | `src/services/botBridge.js`, `src/services/eventBus.js`, `src/services/profileService.js` |
| Interactions | `src/interactions/buttons.js`, `src/interactions/selects.js`, `src/interactions/modals.js` |

---

*From project analysis -- 90+ files, ~32,000 lines of code*
