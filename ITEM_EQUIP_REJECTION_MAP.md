# Item equip rejection and availability map

This document maps the real per-item rejection logic for TF loadout/equip decisions. It intentionally excludes the generic Steam-owner comparisons like `GetOwner() != SteamID` because those are not direct item-level equip rejections.

---

## High-level flow

The loadout UI and equip logic follow this chain:

1. Build candidate item list for a class/slot
2. Reject items that do not match the slot or class
3. Grey out items that conflict with used equip regions
4. Final equip attempt re-checks validity before updating the loadout
5. Loadout validation strips conflicting items after the fact

The main path is:

- [src/game/client/econ/item_selection_panel.cpp](src/game/client/econ/item_selection_panel.cpp#L1000-L1090)
- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L259-L338)
- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L1434-L1490)
- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L1818-L1847)

---

## 1) Candidate list generation: items rejected before they even appear in the slot UI

### Source

- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L294-L338)

### Function

- `CTFInventoryManager::GetAllUsableItemsForSlot()`

### Rejection rules

Each item is rejected if:

- it is not the correct inventory type for the target slot:
  - `bIsAccountIndex != ( pItemData->GetEquipType() == EQUIP_TYPE_ACCOUNT )`
  - file: [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L303-L307)

- it cannot be used by the requested class:
  - `!pItemData->CanBeUsedByClass(iClass)`
  - file: [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L309-L311)

- it is assigned to a different slot for that class:
  - `pItem->GetStaticData()->GetLoadoutSlot(iClass) != iSlot`
  - file: [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L315-L323)

- it is still unpacked / unacknowledged and therefore not ready to be used:
  - `IsUnacknowledged( pItem->GetInventoryPosition() )`
  - file: [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L325-L329)

### Result

Only items that pass all of the above are added to the candidate list for the selected slot.

---

## 2) UI grey-out / reason generation: items are visible but non-selectable

### Source

- [src/game/client/econ/item_selection_panel.cpp](src/game/client/econ/item_selection_panel.cpp#L1132-L1180)
- [src/game/client/econ/item_selection_panel.cpp](src/game/client/econ/item_selection_panel.cpp#L1473-L1499)

### Function

- `CEquipSlotItemSelectionPanel::GetItemNotSelectableReason()`
- `CAccountSlotItemSelectionPanel::GetItemNotSelectableReason()`

### Rejection rules

These return grey-out reasons for an item if:

- it cannot be used by the selected class:
  - `!pItemData->CanBeUsedByClass(m_iClass)`
  - reason: `#Econ_GreyOutReason_CannotBeUsedByThisClass`
  - file: [src/game/client/econ/item_selection_panel.cpp](src/game/client/econ/item_selection_panel.cpp#L1154-L1156)

- it belongs to an incompatible equip slot:
  - `!AreSlotsConsideredIdentical(...)`
  - reason: `#Econ_GreyOutReason_CannotBeUsedInThisSlot`
  - file: [src/game/client/econ/item_selection_panel.cpp](src/game/client/econ/item_selection_panel.cpp#L1158-L1166)

- it conflicts with the currently-used equip regions:
  - `pItemData->GetEquipRegionMask() & unUsedEquipRegionMask`
  - reason: `#Econ_GreyOutReason_EquipRegionConflict`
  - file: [src/game/client/econ/item_selection_panel.cpp](src/game/client/econ/item_selection_panel.cpp#L1170-L1173)

### Notes

This is not “the item is missing from inventory”; it is “the item is valid inventory data but not selectable in this specific slot/class state.”

---

## 3) Final equip attempt: direct rejection before loadout update

### Source

- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L259-L289)

### Function

- `CTFInventoryManager::EquipItemInLoadout()`

### Rejection rules

This returns `false` and prevents equip if:

- item ID is invalid/empty:
  - `iItemID == INVALID_ITEM_ID`
  - [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L262-L266)

- item is not in the local inventory:
  - `m_LocalInventory.GetInventoryItemByItemID( iItemID ) == NULL`
  - [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L268-L270)

- item is not same equip type/slot identity:
  - `!AreSlotsConsideredIdentical(...)`
  - [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L272-L276)

- item cannot be used by target class:
  - `!pItem->GetStaticData()->CanBeUsedByClass( iClass )`
  - [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L278-L281)

### Result

This is the last real client-side gate before `UpdateInventoryEquippedState()` is called.

---

## 4) Slot fetch validation: a loadout-returned item is rejected if it does not belong to that slot

### Source

- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L1434-L1490)

### Functions

- `CTFPlayerInventory::GetItemInLoadout()`
- `CTFPlayerInventory::GetCacheServerItemInLoadout()`

### Rejection rules

When `m_LoadoutItems[iClass][iSlot]` contains an item ID, it only returns that item if:

- `pItem` is not null
- and `AreSlotsConsideredIdentical( pItem->GetStaticData()->GetEquipType(), pItem->GetStaticData()->GetLoadoutSlot(iClass), iSlot )` is true

If not, it falls back to `GetBaseItemForClass( iClass, iSlot )`.

This means the item is effectively treated as unavailable for that slot even if it exists in the inventory.

---

## 5) Equip-region conflict cleanup: invalid loadout item is unequipped after the fact

### Source

- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L1818-L1847)

### Function

- `CTFPlayerInventory::VerifyLoadoutItemsAreValid()`

### Rejection rule

For each loadout slot, it checks:

- `unItemEquipMask = pEquippedItemView->GetItemDefinition()->GetEquipRegionMask();`
- if `unItemEquipMask & unCumulativeRegionMask`, it calls:
  - `InventoryManager()->UpdateInventoryEquippedState( this, INVALID_ITEM_ID, iClass, pEquippedItemView->GetEquippedPositionForClass( iClass ) );`

This strips the conflicting item from the loadout because the item is not valid in the final equipped state.

---

## 6) Empty-slot clearing rejection

### Source

- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L1563-L1588)

### Function

- `CTFPlayerInventory::ClearLoadoutSlot()`

### Rejection rules

It returns `false` if:

- slot index is invalid
- class index is invalid
- the slot is already empty (`LOADOUT_SLOT_USE_BASE_ITEM`)
- `GetItemInLoadout( iClass, iSlot )` returns null

This is a direct per-slot rejection of a slot-clearing action, not a Steam owner check.

---

## 7) Slot identity helper used throughout

### Source

- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L168-L186)

### Function

- `AreSlotsConsideredIdentical()`

This helper is the common gate used for slot compatibility checks:

- `EQUIP_TYPE_CLASS` with misc/building/taunt aliases
- `EQUIP_TYPE_ACCOUNT` with account slots
- otherwise simple slot equality

This is the central “wrong slot / wrong equip target” rule used by the UI and equip logic.

---

## Map

```text
UI slot request
  -> CEquipSlotItemSelectionPanel::UpdateModelPanelsForSelection()
      -> CEquippableItemsForSlotGenerator(...)
          -> TFInventoryManager::GetAllUsableItemsForSlot()
              -> reject if wrong equip type
              -> reject if !CanBeUsedByClass(class)
              -> reject if GetLoadoutSlot(class) != slot
              -> reject if unacknowledged item

Then grey-out / non-selectable checks:
  -> CEquipSlotItemSelectionPanel::GetItemNotSelectableReason()
      -> reject if !CanBeUsedByClass(class)
      -> reject if !AreSlotsConsideredIdentical(...)
      -> reject if equip-region conflict

Final equip attempt:
  -> CTFInventoryManager::EquipItemInLoadout()
      -> reject if !item in local inventory
      -> reject if !AreSlotsConsideredIdentical(...)
      -> reject if !CanBeUsedByClass(class)

Already-equipped item validity:
  -> CTFPlayerInventory::GetItemInLoadout()
      -> reject if current item does not match slot compatibility

Post-equip cleanup:
  -> CTFPlayerInventory::VerifyLoadoutItemsAreValid()
      -> remove item if equip-region mask conflicts
```

---

## Bottom line

The actual per-item rejection logic is not “Steam ownership” checks. It is a combination of:

- inventory membership checks
- class usability checks
- equip-slot compatibility checks
- equip-region conflict checks
- loadout validation cleanup

The most relevant code for patching item availability is:

- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L259-L338)
- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L1434-L1490)
- [src/game/client/econ/item_selection_panel.cpp](src/game/client/econ/item_selection_panel.cpp#L1132-L1180)
- [src/game/shared/tf/tf_item_inventory.cpp](src/game/shared/tf/tf_item_inventory.cpp#L1818-L1847)
