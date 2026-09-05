# Atomic Redis Value Replacement Without Downtime: The Temporary Key Pattern

- Canonical URL: https://imzihad21.github.io/articles/a/atomic-redis-value-replacement-without-downtime-the-temporary-key-pattern-nf2/
- Source URL: https://dev.to/imzihad21/atomic-redis-value-replacement-without-downtime-the-temporary-key-pattern-nf2
- Web View: https://imzihad21.github.io/articles/a/atomic-redis-value-replacement-without-downtime-the-temporary-key-pattern-nf2/
- Published: 2026-04-29T17:41:50.000Z
- Modified: 2026-04-29T17:41:50.000Z
- Reading time: 6 minutes
- Tags: redis, dotnet, distributedsystems, systemdesign

When Redis holds mission‑critical data—authorization rules, feature flags, live configuration—native key updates can cause outages. A simple `DELETE` followed by `SET` opens a window where the key doesn’t exist, and a `HSET` on a live hash can leave stale fields behind. The temporary key pattern uses Redis’s atomic `RENAME` to swap entire data structures in one seamless motion. No gap, no stale leftovers. Let’s see how to implement it in C# with StackExchange.Redis, and later add a distributed lock so even concurrent writers can’t trip each other.

---

### The Problem: Delete First, Set Later

When you need to replace an entire Redis value, the first instinct looks like this:

```csharp
await db.KeyDeleteAsync("config:site");
await db.StringSetAsync("config:site", newJson);
```

Between the two commands the key is gone. Any concurrent reader gets a null (or a default), which for critical data can throw the whole system off. Even worse, if `StringSetAsync` throws, the data is permanently lost until someone runs a backfill job.

For cache‑only data you might tolerate this. For anything that acts as a source of truth, it’s a ticking bomb.

---

### The Merge Trap: Hash and Set Updates That Won’t Go Away

A common attempt to avoid the gap is updating a hash field by field:

```csharp
await db.HashSetAsync("user:123:meta", new HashEntry[] {
    new("role", "user")
});
```

`HSET` only adds or overwrites the fields you pass. It never deletes fields that were present before. If the previous state had `beta_access: true`, it stays—and suddenly a downgraded user still enjoys beta features. The same happens with `Set` if you only add members but never remove the old ones.

You *could* call `KeyDelete` first, but that brings back the empty‑window problem. So we need a way to **completely swap** the data structure without ever exposing an empty or stale version.

---

### The Solution: Write to a Temporary Key, Then Swap

The pattern is simple enough to fit on a sticky note:

1. Write the brand‑new, complete value into a temporary key (e.g., `key:tmp`).
2. Use the `RENAME` command to atomically replace the real key with the temporary one.

`RENAME` is a single Redis command. It deletes the destination if it exists and renames the source in one step. Readers will see either the old, fully correct data, or the new, fully correct data—nothing in between.

We’ll use StackExchange.Redis transactions to bundle the write and the rename together, so a failure during population leaves the original key untouched.

---

### Generic String Replacement Helper

If you store serialized objects as JSON (or any byte blob), here’s a reusable method:

```csharp
public async Task<bool> ReplaceAsync<T>(
    IDatabase db, string key, T value, TimeSpan? expiry = null)
{
    string tempKey = $"{key}:tmp";
    TimeSpan ttl = expiry ?? TimeSpan.FromHours(1);

    var txn = db.CreateTransaction();

    // Write the complete serialized object to temp key
    var setTask = txn.StringSetAsync(
        tempKey, JsonSerializer.Serialize(value), ttl);

    // Atomically rename: this is the magic
    var renameTask = txn.KeyRenameAsync(tempKey, key);

    bool committed = await txn.ExecuteAsync();
    await Task.WhenAll(setTask, renameTask);

    return committed;
}
```

You call it like this:

```csharp
var myConfig = new { Theme = "dark", CacheTimeout = 30 };
bool ok = await ReplaceAsync(db, "app:config", myConfig);
```

No need to delete, no chance for an empty read. If the transaction fails, `key` remains completely untouched and you can retry.

---

### Why a Transaction Matters

You might ask: can’t I just chain `StringSetAsync` and `KeyRenameAsync` without `CreateTransaction`? Technically yes, but if the `StringSetAsync` succeeds and the `KeyRenameAsync` fails (network blip), the `:tmp` key is left dangling and the real key still holds old data. That’s not disastrous, but it’s a cleanup headache. A transaction ensures both commands are queued together (`MULTI/EXEC`)—they either both happen or both don’t. Atomicity at the network round‑trip level.

Note: The transaction doesn’t isolate reads from outside clients. But because `RENAME` itself is a single atomic command, the viewer never sees the intermediate `:tmp` key in the real key’s place. The old value transparently turns into the new one.

---

### Replacing a Set the Same Way

Need to refresh a set of allowed IPs or permission identifiers? Use exactly the same pattern:

```csharp
public async Task ReplaceSetAsync(
    IDatabase db, string key, string[] members)
{
    string tempKey = $"{key}:tmp";

    var txn = db.CreateTransaction();

    // Clean any leftover temp key from a previous crash
    txn.KeyDeleteAsync(tempKey);

    // Add all new members to the temp set
    txn.SetAddAsync(tempKey, members.Select(m => (RedisValue)m).ToArray());

    // Atomically swap
    txn.KeyRenameAsync(tempKey, key);

    await txn.ExecuteAsync();
}
```

No members get lost, no members stay behind that should have been removed. The set is replaced in one instant.

```csharp
await ReplaceSetAsync(db, "ip:allowlist", new[] { "10.0.0.1", "10.0.0.2" });
```

---

### Replacing a Hash: Fields Come and Go Together

This is the fix for the stale‑admin story (you know the one). Instead of calling `HashSet` on the live key, do this:

```csharp
public async Task ReplaceHashAsync(
    IDatabase db, string key, HashEntry[] entries)
{
    string tempKey = $"{key}:tmp";

    var txn = db.CreateTransaction();
    txn.KeyDeleteAsync(tempKey);
    txn.HashSetAsync(tempKey, entries);
    txn.KeyRenameAsync(tempKey, key);

    await txn.ExecuteAsync();
}
```

Now every old field disappears because you replace the entire hash.

```csharp
await ReplaceHashAsync(db, "user:123:meta", new HashEntry[] {
    new("role", "user")
    // beta_access is gone – intentionally
});
```

---

### Redis Cluster? Keep the Keys Together

If you run Redis Cluster, `RENAME` only works when source and destination are in the same hash slot. Use curly‑brace hash tags to enforce that:

```csharp
string mainKey = $"{{user:{userId}}}:permissions";
string tempKey = $"{{user:{userId}}}:permissions:tmp";
```

Redis calculates the slot based on the substring inside the `{}`. Both keys end up on the same node, and the rename works without a cross‑slot error.

---

### When Things Don’t Go as Planned

Even an atomic pattern needs a safety net.

- **Transaction fails (`ExecuteAsync` returns false):** The original key is untouched. Just retry. If the temp key already existed (a previous crash), the `KeyDeleteAsync` inside the transaction clears it first.
- **Crash after `:tmp` is written but before rename:** The temp key hangs around. The `expiry` on the string helper will clean it automatically after TTL. For sets and hashes you can add an explicit `KeyExpireAsync(tempKey, TimeSpan.FromMinutes(5))` inside the transaction or run a janitor that deletes keys matching `*:tmp` that are older than a few minutes.
- **Multiple writers racing on the same key:** The rename pattern alone doesn’t prevent two processes from both writing to `:tmp` and trying to rename. One will overwrite the other’s work, and there’s no guarantee which one survives. We can fix this with a distributed lock.

---

### Taking It Further with a Distributed Lock

If more than one process can trigger a replacement for the same key—like two admin panels or concurrent background jobs—you need a mutex to serialize the operation. StackExchange.Redis provides built‑in primitives for that: `LockTake` and `LockRelease`.

Here’s a generic locking helper that executes any action under a Redis‑based lock:

```csharp
public async Task<bool> ExecuteWithLockAsync(
    IDatabase db, string lockKey, TimeSpan lockExpiry, Func<Task> action)
{
    string token = Guid.NewGuid().ToString("N");
    bool acquired = await db.LockTakeAsync(lockKey, token, lockExpiry);

    if (!acquired)
        return false;

    try
    {
        await action();
        return true;
    }
    finally
    {
        // Release the lock – fire‑and‑forget is enough here
        await db.LockReleaseAsync(lockKey, token, CommandFlags.FireAndForget);
    }
}
```

The `token` ensures only the locker can release the lock. If the process crashes, the lock auto‑expires after `lockExpiry`, preventing deadlocks.

Now wrap any of the `Replace*` methods. For example, safe set replacement with a lock:

```csharp
public async Task SafeReplaceSetAsync(
    IDatabase db, string key, string[] members)
{
    string lockKey = $"lock:{key}";
    TimeSpan lockExpiry = TimeSpan.FromSeconds(10);

    bool lockAcquired = await ExecuteWithLockAsync(
        db, lockKey, lockExpiry, () => ReplaceSetAsync(db, key, members));

    if (!lockAcquired)
        throw new Exception("Could not acquire lock for replacement");
}
```

What we gain:

- **Serialized writers:** Only one updater touches the temp key at a time. No more last‑writer‑wins anarchy.
- **Complete safety:** The lock is held only around the write + rename transaction, not during data preparation.
- **No external packages:** Everything uses the same `IDatabase` instance and standard Redis commands.

You can apply the same wrapper to `ReplaceHashAsync` or `ReplaceAsync<T>`. Just pick a lock expiry longer than your worst‑case operation time.

---

### This Trick Is Old and Everywhere

The idea of “write to a temp file and rename” is used by Linux package managers, file systems, and even Redis internally for its own persistence. It’s not a workaround—it’s the standard way to atomically replace something that can’t be replaced in‑place. Adding a lock turns a great trick into a bulletproof asset.