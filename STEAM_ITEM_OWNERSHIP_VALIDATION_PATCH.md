# Steam item ownership validation: what it is and how to patch it out for this mod

## Summary

This project does not use a single dedicated “Steam owns this item?” function. Instead, it validates ownership by comparing inventory/account identities against the logged-in Steam account and by gating loadout processing behind those comparisons.

For a mod that wants unrestricted/infinite item access, the cleanest patch is to remove or bypass the local-owner checks in the inventory/loadout paths and allow the local client to process items regardless of whether `GetOwner()` matches the active Steam account.

---

## Ownership model in this codebase

The base ownership data is stored on each item as an account ID:

- [src/game/shared/econ/econ_item.h](src/game/shared/econ/econ_item.h#L307-L308)
- [src/game/shared/econ/econ_item.h](src/game/shared/econ/econ_item.h#L547-L547)

The inventory manager resolves inventories by account ID:

- [src/game/shared/econ/econ_item_inventory.cpp](src/game/shared/econ/econ_item_inventory.cpp#L191-L200)

Steam inventory requests are also validated up front to ensure the target SteamID is a valid individual account:

- [src/game/shared/econ/econ_item_inventory.cpp](src/game/shared/econ/econ_item_inventory.cpp#L141-L171)

This means the project treats ownership as a Steam account identity problem, not a purely local `item_id` problem.

---

## Exact validation points found

### 1) Local loadout load is blocked if the inventory owner does not match the local Steam user

File:
- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L902-L915)

Relevant logic:

```cpp
if (GetOwner() != steamapicontext->SteamUser()->GetSteamID())
    return;
```

This prevents the local loadout from loading when the inventory owner is not the currently logged-in Steam account.

### 2) Local save/loadout persistence is similarly gated

File:
- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L971-L999)

Relevant logic:

```cpp
if (GetOwner() != steamapicontext->SteamUser()->GetSteamID())
    return;
```

This denies local file save/load behavior for nonlocal inventories.

### 3) Loadout verification is skipped for other players and for mismatched owners

File:
- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L1850-L1879)

Relevant logic:

```cpp
if ( GetOwner() != steamapicontext->SteamUser()->GetSteamID() )
    return;
```

This is a direct ownership guard before the game verifies the validity of changed loadout items.

### 4) The local inventory is only treated as self-owned when the owner SteamID matches

File:
- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L1268-L1288)

Relevant logic:

```cpp
if ( InventoryManager()->GetLocalInventory() == this && GetOwner() == steamIDOwner )
```

This is how the code decides whether a SOCreated event belongs to the local player’s own inventory.

### 5) The item system tracks ownership as account ID transfer state

File:
- [src/game/shared/econ/econ_item.cpp](src/game/shared/econ/econ_item.cpp#L1011-L1051)

This runs on ownership transfer and marks item state accordingly.

---

## Why this matters for a mod

If you want this mod to behave like a sandbox or cheat inventory, these checks block exactly the behavior you need to remove:

- local loadout being ignored
- item validation refusing to process mismatched owners
- inventory ownership being treated as a hard Steam identity gate
- GC/local inventory updates only accepting self-owned items

In other words, this is the main “you do not own this item” enforcement path in the project.

---

## Patch-out strategy for this mod

### Recommended minimal patch

Patch the local-owner guard out in the main TF inventory validation points:

1. Remove the early returns in:
   - [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L902-L915)
   - [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L971-L999)
   - [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L1850-L1879)

2. Replace them with a mod condition such as:

```cpp
#ifdef SCRAPWAVE_UNRESTRICTED_ITEMS
    // allow local processing even when inventory owner is not the active Steam account
#else
    if (GetOwner() != steamapicontext->SteamUser()->GetSteamID())
        return;
#endif
```

This keeps the original behavior intact for stock builds while enabling unrestricted behavior for the mod build.

### Safer variant: add a single feature flag

For a cleaner patch, implement a project-wide toggle and use it in all ownership checks:

```cpp
static bool IsInventoryOwnershipValidationDisabled()
{
#ifdef SCRAPWAVE_UNRESTRICTED_ITEMS
    return true;
#else
    return false;
#endif
}
```

Then wrap each guard with:

```cpp
if ( !IsInventoryOwnershipValidationDisabled() && (GetOwner() != steamapicontext->SteamUser()->GetSteamID()) )
    return;
```

This is preferable because it is easy to disable for testing and easy to remove later.

---

## More aggressive patch option

If the goal is to fully bypass ownership validation rather than just local inventory logic, then also consider removing the account-based filtering in the inventory manager itself:

- [src/game/shared/econ/econ_item_inventory.cpp](src/game/shared/econ/econ_item_inventory.cpp#L191-L200)

That layer is where an account is searched by owner ID, and it is another direct ownership enforcement point.

For a “modded unrestricted inventory,” the more complete approach is:

- keep the inventory data readable
- ignore Steam account ownership mismatches during loadout processing
- do not reject local inventory requests based on `GetOwner() != currentSteamID`
- do not abort validation for nonself inventory states on the local client

---

## What not to do

Avoid a broad patch that removes all inventory logic. The safer approach is to bypass only the ownership gate, not the whole item system.

Do not blindly remove:

- the `CPlayerInventory` lookup logic entirely
- item ID / schema validation
- actual inventory serialization

Those are still required for the item system to function. The ownership gate is the part to disable.

---

## Recommended mod-specific final patch shape

For this project, I would patch it as follows:

- Add a compile-time mod flag: `SCRAPWAVE_UNRESTRICTED_ITEMS`
- Wrap the Steam-owner checks in the TF inventory code with that flag
- Leave the rest of the econ/item logic untouched
- Keep stock behavior as default for any nonmod build

That gives a clean, reversible change without harming the normal game logic outside the mod build.

---

## Files to patch

Primary patch sites:

- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp)
- [src/game/shared/econ/econ_item_inventory.cpp](src/game/shared/econ/econ_item_inventory.cpp)
- optional global gateway if you want a single toggle:
  - [src/game/shared/econ/econ_item.h](src/game/shared/econ/econ_item.h)
  - [src/game/shared/econ/econ_item.cpp](src/game/shared/econ/econ_item.cpp)

---

## Bottom line

The real gate is the Steam-account identity comparison. The project enforces ownership by checking whether `GetOwner()` equals the active Steam account before processing local inventory/loadout data.

For this mod, the practical fix is to disable or bypass those owner checks in the TF inventory path while leaving the rest of the item system intact.
