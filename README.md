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
16. [Technology Deep Dive](#16-technology-deep-dive)

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

---

## 16. Technology Deep Dive

### 16.1 AI Orchestration Engine (`groqChat.js` -- 1685 lines)

The central AI system that processes every user message through a multi-stage pipeline.

**Core Loop:**
```
User Message -> Dedup Check -> Guild Approval -> Message Classification
    -> Context Building -> LLM Chain Call -> Response Parsing -> Reply
    -> Background Memory Extraction (async)
```

**Message Classification (`classifyComplexity`):**
Returns complexity tier 0-4 based on keyword analysis:
- **Tier 0**: Greetings, short banter (< 25 chars, matches greeting regex)
- **Tier 1**: Standard questions, general chat (default)
- **Tier 2**: Coding keywords, > 300 chars, multi-step questions
- **Tier 3**: Deep keywords (architecture, math, refactoring), > 800 chars
- **Tier 4**: Extreme keywords (agent, debate, audit), > 2500 chars

**Advanced Intent Analysis (`analyzeMessageIntent`):**
Single-pass LLM classification that outputs JSON with:
- `category`: coding | math | creative | roleplay | general
- `emotion`: happy | angry | sad | confused | joking | serious
- `routing_mode`: FAST | DEEP | MEMORY | TOOL | DEBATE | SEARCH
- `context_budget`: Dynamic token budgets for history, memory, profile
- `confidence`: 0-1 confidence score

**Context Building (`buildOptimizedContext`):**
Assembles prompt context within token budgets:
- Profile summary (IGN, sect, weapons, inner ways) -- 200 token budget
- Personality block (communication style, tone, relationship tier)
- User memory facts ranked by relevance (500 token budget)
- Server memory facts ranked by relevance
- Channel summary for conversation continuity
- RPG game stats if player exists

**Memory Ranking Algorithm (`rankFacts`):**
Hybrid semantic ranking with exponential time decay:
```
score = (keywordScore * 0.4) + (semanticScore * 0.6) + (importance * 0.2 * decay) + recencyBoost
```
- **Keyword score**: +10 per exact word match
- **Semantic score**: +10 per synonym match (via CONCEPT_MAP), or bigram similarity * 10 if > 0.6
- **Time decay**: `exp(-ageDays * decayRate)` where rate varies by type:
  - identity: 0 (never decays)
  - preference: 0.005
  - skill: 0.01
  - temporary: 0.1
- **Correction boost**: Facts starting with "Correction:" get +100 score

**Bigram Similarity (`bigramSimilarity`):**
Character-level Jaccard similarity using 2-gram sets:
```
similarity = |bigrams(a) INTERSECTION bigrams(b)| / |bigrams(a) UNION bigrams(b)|
```

**Synonym Detection (`areSynonyms`):**
Concept maps group related terms (e.g., {fps, shooter, shooting}, {coding, developer, code}).

**Response Parsing (`parseBrainResponse`):**
Extracts structured data from LLM XML output:
- `<action>`: TEXT_ONLY | TEXT_AND_IMAGE | IMAGE_ONLY
- `<image_prompt>`: Image generation prompt
- `<memory_update>`: Memory commands (ADD/UPDATE/DELETE)
- `<reply>`: Actual response text

**Token Budget Enforcement (`enforceTokenBudget`):**
Trims oldest messages from history until total estimated tokens fit within limit. Uses `estimateTokens()` which approximates at 4 chars/token.

**Memory Update Debounce:**
Per-user 3-minute cooldown between memory extraction runs. Stale map entries auto-pruned every 30 minutes.

---

### 16.2 Model Router & Cascade (`modelRouter.js` -- 652 lines)

Intelligent model selection and API key rotation system.

**8-Model Chain:**
```
llama-3.1-8b-instant -> qwen/qwen3-32b -> llama-3.3-70b-versatile
    -> openai/gpt-oss-120b -> openai/gpt-oss-20b
    -> groq/compound-mini -> groq/compound -> allam-2-7b
```

**Per-Model Rate Limits (hardcoded):**
| Model | TPM | RPD | RPM |
|-------|-----|-----|-----|
| llama-3.1-8b-instant | 6,000 | 14,400 | 30 |
| qwen/qwen3-32b | 6,000 | 1,000 | 60 |
| llama-3.3-70b-versatile | 12,000 | 1,000 | 30 |
| groq/compound-mini | 70,000 | 250 | 30 |
| meta-llama/llama-4-scout | 30,000 | 1,000 | 30 |

**Smart Model Selection (`selectAndSortModels`):**
Sorts models by composite score:
1. TPM fits required tokens (priority)
2. Not near 80% TPM limit (priority)
3. Performance score from feedback loop (descending)
4. Original chain position (ascending)
5. RPD remaining (descending)

**Per-Minute Token Tracking (`modelTokenUsage` Map):**
- Sliding 60-second window per model
- Auto-reset when window expires
- Reports usage percentage vs TPM limit
- 80% threshold marks model as "near limit"

**Key Pool Management (`getAPIKeysPool`):**
Assembles key pool from 3 sources (deduplicated by value):
1. Guild-specific key (from guild config, highest priority)
2. Community-contributed keys (from `contributed_keys` table)
3. Default system key (`GROQ_API_KEY` env var)

**Key Selection Scoring (`selectBestKey`):**
```
score = (rateLimited * 1000) + (failures * 10) + (lastUsed / 1000000)
```
Lowest score wins. Rate-limited keys get huge penalty. Recently used keys get slight penalty for rotation.

**Rate Limit Handling:**
- Parses `x-ratelimit-reset-tokens` header (e.g., "1m30s", "250ms")
- Marks key rate-limited for reset time + 5s buffer (max 65s)
- 401 Unauthorized: Auto-disables contributed key permanently
- TCP errors: Key NOT penalized (network issue, not key issue)
- 429 rate limit: Waits reset time, then tries next key

**Semaphore (`requestSemaphore`):**
Max 5 concurrent API requests. Queue max 100. Rejects with error when full.

**Chain Execution (`callGroqAPIWithChain`):**
For each model in sorted order:
  For each key in pool (excluding already-tried):
    Try request -> success: return | failure: mark key, try next
  If all keys exhausted for this model: try next model
Last resort: Try near-limit models if all healthy models exhausted

**Token Usage Correction:**
After response, if Groq returns actual `usage.total_tokens`, corrects the estimated count. Also checks `x-ratelimit-remaining-tokens` header to sync with Groq's server-side tracking.

**Prompt Cache Detection:**
Logs `cache_read_input_tokens` from Groq response (50% discount applied by Groq for cached prompts).

---

### 16.3 Message Classification System (`messageClassifier.js` -- 356 lines)

Multi-tier classification for routing messages to appropriate models.

**Capacity Calculation (`calculateCapacity`):**
Determines max intelligence tier based on available resources:
- 4+ unique key accounts AND 4+ healthy models -> tier 4
- 3+ accounts AND 3+ models -> tier 3
- 2+ accounts AND 2+ models -> tier 2
- Otherwise -> tier 1

**Intelligence Tier Resolution (`determineIntelligenceTier`):**
```
tier = min(complexity, maxAllowedTier)
tier = max(1, tier)  // minimum tier 1
```
Exception: Tier 0 (greetings) always stays tier 0.

**Racing Disable (`shouldDisableRacing`):**
Disables multi-model racing when primary or smart model usage exceeds 50% daily.

**LLM-Powered Intent Analysis:**
Full prompt analysis using `llama-3.1-8b-instant` with `temperature: 0` and `response_format: json_object` for deterministic classification output.

---

### 16.4 Critic Passes & Multi-Model Debate (`criticPasses.js`)

Optional multi-model pipeline for complex queries:
- **Architect Model**: Generates initial response
- **Critic Model**: Reviews and critiques the response
- **Judge/Critic Model**: Arbitrates between architect and critic
- **Finalizer**: Produces final polished output
- **Alternative Thinker**: Provides different perspective
- **Speed Runner**: Quick low-resource response

Models are selected from the chain based on availability and health.

---

### 16.5 Resource Manager (`resourceManager.js` -- 418 lines)

Event-driven system monitoring with adaptive response.

**Sampling:**
- CPU: 1-second interval using `process.cpuUsage()` delta
- RAM: 2-second interval using `process.memoryUsage()`
- Normalized to 0-100% across all cores

**Zone Determination:**
```
emergency: cpu >= 90% OR rss >= 95% OR heap >= 95%
high:      cpu >= 80% OR rss >= 80% OR heap >= 80%
busy:      cpu >= 70% OR rss >= 70% OR heap >= 70%
normal:    everything below
```

**Hysteresis (Debounce):**
Zone changes debounced to 3-second minimum interval to prevent rapid bouncing (e.g., high -> busy -> high).

**Adaptive Queue Delay:**
```
normal: base delay (10ms)
busy:   max(5ms, base * 2)
high:   max(10ms, base * 4)
emergency: max(20ms, base * 8)
```

**Adaptive Worker Count:**
Dynamically adjusts worker threads based on zone:
- normal: minWorkers
- busy: minWorkers + 1
- high: minWorkers + 2
- emergency: maxWorkers

**Event Emissions:**
- `zoneChange(newZone, oldZone)` -- Triggers cache clearing, scheduler throttling
- `emergencyChange(isEmergency)` -- Emergency protocol activation
- `tickEmergency` -- 5-second interval during emergency state

---

### 16.6 Encryption System (`crypto.js` -- 144 lines)

AES-256-GCM authenticated encryption for data at rest.

**Two Independent Keys:**
1. `ENCRYPTION_SECRET` -- For API key encryption (64-char hex = 32 bytes)
2. `SYNC_ENCRYPTION_KEY` -- For sync payload encryption (64-char hex = 32 bytes)

**String Encryption (`encryptApiKey`):**
```
Storage format: <iv_hex>:<authTag_hex>:<ciphertext_hex>
IV: 12 bytes (96-bit, recommended for GCM)
Auth Tag: 16 bytes (128-bit)
Algorithm: AES-256-GCM
```

**Buffer Encryption (`encryptBuffer`):**
Binary format: `[IV(12)][AuthTag(16)][Ciphertext]` -- prepended to buffer.

**Legacy Handling:**
- `PLAIN:` prefix: Transparent plaintext passthrough
- No colons: Assumed raw plaintext from pre-encryption era
- Graceful degradation: Logs warning but doesn't crash if key missing

---

### 16.7 Database Caching Layer (`db.js` -- 899 lines)

Multi-tier caching with LRU eviction and memory pressure response.

**Cache Configuration:**
```javascript
_profileCache: LRUCache({ max: 500, ttl: 10 * 60 * 1000 })  // 500 entries, 10 min
_guildCache:   LRUCache({ max: 500, ttl: 10 * 60 * 1000 })  // 500 entries, 10 min
_welcomeDmCache: Map({ max: 10000, ttl: 24 * 60 * 60 * 1000 }) // 24h
```

**Memory Pressure Handler (`_onMemoryPressure`):**
Triggered by Resource Manager on `high` or `emergency` zone:
1. Clears `_profileCache` entirely
2. Clears `_guildCache` entirely
3. Trims `_welcomeDmCache` to half max
4. Logs RSS freed

**Deep Clone Safety:**
All cache reads return `JSON.parse(JSON.stringify(obj))` clones to prevent external mutation.

**Welcome DM Cache:**
Hourly cleanup of expired entries (24h TTL). Max 10,000 entries enforced by LRU-style eviction.

---

### 16.8 Turso Embedded Replica System (`turso.js` -- 151 lines)

Cloud-synced SQLite with local fallback.

**Architecture:**
```
Local SQLite (WAL mode) <-> libsql Embedded Replica <-> Turso Cloud
```

**Monthly Write Limit:**
Tracks writes via `system_stats` table. Hard cap at 9,999,000 (safety buffer below 10M limit). When reached, sync suspended.

**Cached Reads:**
```javascript
tursoReadCache: LRUCache({ max: 500, ttl: 5 * 60 * 1000 })
```
Invalidated by pattern matching on SQL queries.

**Database Modes:**
- `turso`: Normal embedded replica with cloud sync (default)
- `local`: Local SQLite only, no cloud sync
- `standalone_turso`: Direct Turso cloud without local SQLite

**Sync Handling:**
libsql native sync handles replication automatically. Manual sync function is a no-op (legacy compatibility).

---

### 16.9 Combat System (`combat.js` -- 909 lines)

Turn-based RPG combat with stat calculations and dual-weapon swapping.

**Player Stat Formulas:**
```
maxHp      = body * 15 + defense * 5 + sum(equipment.stat_hp) + sum(tuning.hp)
maxAtk     = power * 3.5 + sum(equipment.stat_atk_max) + sum(tuning.atk)
minAtk     = maxAtk * (0.4 + agility / 200)
defMit     = defense * 1.2 + sum(equipment.stat_def) + sum(tuning.def)
critRate   = min(80, agility / 5) + sum(tuning.crit)
affRate    = min(50, momentum / 8) + sum(tuning.aff)
critDamage = 1.35 (35% base)
```

**Sect Passive Modifiers:**
Applied as multiplicative buffs (e.g., `damageMult`, `defenseMult`, `critRateBonus`, `lifesteal`, `healBoost`, `startingShield`).

**Enemy Scaling:**
```
scale = 1 + (level - 1) * 0.15
hp    = floor(BASE_HP * scale * eliteMultiplier)
atk   = floor(BASE_ATK * scale * eliteMultiplier)
def   = floor(BASE_DEF * scale * eliteMultiplier)
```
Elite multiplier: HP x2.5, ATK x1.5, DEF x1.3

**Turn Resolution (`executeCombatTurnCalc`):**

Player attack type selection:
```
60% chance: Check crit -> if crit: critDamage multiplier
            else: Precision Hit (normal)
40% chance: Check affinity -> if aff: 1.20x multiplier
            else: Normal Hit
```

Action multipliers:
- Martial: 1.5x
- Special: 1.2x (+ 25% crit chance)
- Charged: 2.0x (- 20% defense)
- Block: 0.5x (+ 50% counter damage, - 50% incoming)
- Mystic Skills: Fixed damage formulas (e.g., Tai Chi: `80 + rank*15 + tier*50`)

**Weapon Passives:**
- Sword: 25% chance + 20% bonus damage (Relentless Chase)
- Spear: 30% chance + 30% bonus damage (Guard Break)
- Fan: + 5% maxHp healing per turn
- Umbrella: + 25 shield per turn
- Twin Blades: + 0.2 crit damage multiplier on crits
- Rope Dart: 20% chance + 40% bonus damage (Thread Combo)

**Dual-Weapon Swap:**
Every 3rd phase, automatically swaps between primary and secondary weapon.

**Mystic Skills (6 skills):**
- **Tai Chi** (35v): `80 + rank*15 + tier*50` damage, heal 10% maxHp
- **Meridian Touch** (45v): Base damage, stun enemy, -defense debuff
- **Celestial Seize** (40v): 1.2x base damage, steal enemy attack
- **Golden Body** (50v): Shield + 50% defense boost for 2 turns
- **Cloud Steps**: Evasion skill
- **Glow of Fireflies**: Damage over time

---

### 16.10 Loot Generation (`loot.js` -- 251 lines)

Probabilistic item generation with rarity tiers and stat rolling.

**Rarity Weights:**
| Rarity | Drop Chance | Stat Multiplier | Tuning Slots |
|--------|-------------|-----------------|--------------|
| Common | 25% | 0.8x | 1 |
| Uncommon | 30% | 1.0x | 2 |
| Rare | 25% | 1.25x | 3 |
| Epic | 12% | 1.5x | 4 |
| Legendary | 5% | 1.9x | 5 |
| Mythic | 2% | 2.4x | 5 |
| Divine | 1% | 3.0x | 5 |

**Stat Generation:**
```
finalStat = floor(baseStat * rarityMultiplier * (1 + level * 0.1))
```

**Item Type Distribution:**
- 35% weapon (random type from 6)
- 40% armor (random slot: head/chest/arms/legs)
- 25% accessory (random slot: pendant/disc/ring)

**Dismantle Rewards:**
| Rarity | Gold |
|--------|------|
| Common | 50 |
| Uncommon | 100 |
| Rare | 250 |
| Epic | 600 |
| Legendary | 1,500 |
| Mythic | 3,500 |
| Divine | 10,000 |

---

### 16.11 Raid Engine (`raidEngine.js` -- 505 lines)

Multiplayer boss encounters with scaling and threat management.

**Boss Encounters:**
| Boss | HP | ATK | DEF | Mechanics |
|------|-----|-----|-----|-----------|
| Dalang (Demon Steed) | 1,800 | 36 | 8 | Mobile, leaped stomps, Evade to dodge |
| Wandering Swordsman | 1,500 | 45 | 12 | Precision parrier, Parry to stagger |
| Corrupted General | 2,600 | 32 | 22 | Armor behemoth, Shield to mitigate |

**Level Scaling:**
```
scaledLevel = getScaledBossLevel(playerLevel)  // brackets: 5, 15, 25, 40, 60
scaleMult   = sqrt(scaledLevel)
bossMaxHp   = floor(baseHp * scaleMult * teamMultiplier)
bossAtk     = floor(baseAtk * (1 + scaledLevel * 0.08))
```

**Team Size Multiplier:**
- 5-player lobby: 1.0x boss HP
- 10-player lobby: 1.8x boss HP

**Threat System:**
Each player accumulates threat from damage dealt. Boss targets highest threat player. Tanking mechanics reduce incoming damage.

**In-Memory State:**
Raids stored in `activeRaids` Map with automatic cleanup:
- Victory/defeat states: cleaned after 60 minutes
- Lobby state: cleaned after 60 minutes of inactivity
- Cleanup runs every 10 minutes via Resource Manager scheduler

---

### 16.12 Story Engine (`storyEngine.js` -- 955 lines)

LLM-driven dynamic narrative with pre-generated branches.

**Jaro-Winkler Similarity:**
Used for fuzzy matching player text input to story choices:
```
matchWindow = floor(max(len1, len2) / 2) - 1
Jaro = (matches/len1 + matches/len2 + (matches-transpositions)/matches) / 3
JaroWinkler = Jaro + l * p * (1 - Jaro)  // p = 0.1, l = common prefix length (max 4)
```

**Story Generation Flow:**
1. Player selects root story node (predefined)
2. LLM generates 2-3 choices for each node
3. Choices are pre-generated in background (Jaro-Winkler matching)
4. Player text input matched to closest choice via similarity
5. Cloudflare AI generates image for each story node

**Image Generation:**
Uses Cloudflare Workers AI (`@cf/stabilityai/stable-diffusion-xl-base-1.0`) with Pollinations.ai fallback.

---

### 16.13 Background Thinking System (`backgroundThinking.js` -- 363 lines)

Async post-conversation analysis and memory consolidation.

**Trigger:**
Queue-based: Runs every 30 seconds, processes channels quiet for 2+ minutes.

**Analysis Prompt (4 outputs):**
1. **Feedback/Sentiment**: positive | negative | neutral + explanation
2. **Personality Patch**: tone, depth, nickname, topics, knowledge, avoidances, relationship metrics
3. **Memory Updates**: ADD/UPDATE/DELETE commands for user/server facts
4. **Channel Summary**: Consolidated conversation summary (< 200 chars)

**JSON Repair (`repairTruncatedJSON`):**
Handles truncated LLM output by:
1. Closing unclosed strings
2. Removing trailing commas
3. Closing unclosed braces/brackets via stack tracking

**Model Score Feedback Loop:**
- Positive sentiment: +0.05 score delta per model used
- Negative sentiment: -0.10 score delta per model used
- Scores influence model selection in `selectAndSortModels`

**Memory Update Commands:**
```
ADD_USER_FACT: <fact>
UPDATE_USER_FACT: <idx> | <new_fact>
DELETE_USER_FACT: <idx>
ADD_SERVER_FACT: <fact>
UPDATE_SERVER_FACT: <idx> | <new_fact>
DELETE_SERVER_FACT: <idx>
```

**Correction Learning:**
Detects user corrections in history (e.g., "no that's wrong") and stores failure reasons as "Correction:" facts with +100 priority boost.

---

### 16.14 Profile Card System (`nameplate.js` -- 315 lines)

Discord embed-based profile cards with tier color coding.

**Tier Color Mapping:**
| Inner Way Tier | Color | Hex |
|----------------|-------|-----|
| T6 | Crimson | 0xe74c3c |
| T5 | Orange | 0xe67e22 |
| T4 | Gold | 0xf1c40f |
| T3 | Purple | 0x9b59b6 |
| T2 | Blue | 0x3498db |
| T1 | Green | 0x2ecc71 |
| T0 | Grey | 0x95a5a6 |

**Nameplate Components:**
- Header (sect/guild, registration status)
- Player info (Discord mention, IGN)
- Bio/personal quote (smart-truncated at word boundaries)
- Equipped loadout (primary/secondary weapons)
- 4 Inner Way slots with tier/type indicators
- Relationship status (partners/disciples)
- Leave/vacation status

**SVG-to-PNG Pipeline:**
Worker thread pool (`imageWorkerPool.js`) processes SVG templates via `sharp` for image-based cards.

---

### 16.15 State Machine (`stateManager.js` -- 592 lines)

Player game state management with Discord UI rendering.

**States:**
```
town -> explore_select -> combat
town -> shop -> town
town -> training -> town
town -> develop -> skills/breakthrough
town -> campaign -> story
town -> raid_select -> raid_lobby -> raid_combat
```

**UI Rendering:**
Each state renders Discord embeds with:
- Thumbnail images (pre-generated PNGs from `generated_art/`)
- Interactive button rows for navigation
- Status fields (HP, gold, realm, level)

**Action Lock System (`gameStateLock.js`):**
Per-user mutex preventing concurrent game actions. Uses `acquireActionLock`/`releaseActionLock` pattern.

---

### 16.16 WebSocket & Live Updates

**Dashboard Communication:**
- WebSocket server in Express for real-time PM2 log streaming
- `botBridge.js` sends commands from dashboard to bot process
- `eventBus.js` internal pub/sub for cross-process events

**Live Console:**
WebSocket streams PM2 logs to dashboard admin panel in real-time.

---

### 16.17 Image Processing Pipeline

**Worker Pool (`imageWorkerPool.js`):**
- Configurable min/max workers (1-4)
- SVG-to-PNG conversion via `sharp`
- Pre-generated thumbnails for game locations
- Story node image generation via Cloudflare AI

**Generated Art Assets:**
Pre-rendered PNG thumbnails in `src/game/generated_art/`:
- Town square, shop, world map
- Region thumbnails (Qinghe Forest, Kaifeng Outskirts)
- Enemy portraits

---

### 16.18 Maintenance Scheduler (`maintenance.js` -- 209 lines)

Daily automated tasks:
- API key validation (test each contributed key)
- Vacation check-in processing
- Active member thread synchronization
- Turso database backup
- Stale cache cleanup
- Memory consolidation ("bot dreams")

---

### 16.19 Bot Dreams (`botDreams.js` -- 222 lines)

Nightly memory consolidation system:
- Deduplicates similar memory facts
- Merges related facts
- Prunes low-importance, unused facts
- Consolidates personality data
- Runs during low-traffic hours

---

*From project analysis -- 90+ files, ~32,000 lines of code*
