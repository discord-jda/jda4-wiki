# Guild Streaming

Whenever a new JDA instance is created or an active session needs to be restarted (invalidated) all guilds will be added to an internal setup system. When this happens they go through the following stages:

## 0) Unavailable

When a bot account first starts up it receives all connected guilds (in the shard) with `unavailable: true`. Discord does this to prevent gigantic `READY` payloads, the guilds will be sent afterwards using `GUILD_CREATE` events.

## 1) Initialize

For all account types this will be the initial state a guild acquires when being added. Here we decide
where we should [sync](#2-guild-sync), [chunk](#3-chunking), or [ready](#4-ready). In this stage we just store the properties we received. Note that bot accounts will only enter this stage when the guild becomes available through `GUILD_CREATE`. When no `GUILD_CREATE` is received discord will instead inform us with a `GUILD_DELETE` that we should not wait for this guild and it will be marked unavailable and ignored for setup.

## 2) Guild Sync

Client accounts require an additional setup step because they are meant to only receive events when loaded (client efficiency). However to function properly as a bot we need to request a sync for them first.
Once we receive the requested `GUILD_SYNC` we continue with [chunking](#3-chunking) or [ready](#4-ready)

## 3) Chunking

When a guild is bigger it might now have all members in the initial payload which means we have to request additional members through a chunk request. Once all members are chunked we continue and [ready](#4-ready)

## 4) Ready

Now all properties including members have been loaded so we can finally create the Guild instance and add it to the cache. If the guild was from the initial READY we need to fire a `GuildReadyEvent` otherwise it was joined during runtime and we fire a `GuildJoinEvent` instead.

If all guilds are now ready we fire a `ReadyEvent` to inform our client. (in this case we ignore guilds that became unavailable during streaming)

### Notes

If any guild becomes unavailable during these stages it is removed entirely from the setup process and ignored until it becomes available again through a `GUILD_CREATE` with `unavailable: false`.