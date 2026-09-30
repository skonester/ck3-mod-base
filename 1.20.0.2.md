# Crusader Kings III — 1.19.0.6 "Scribe" → 1.20.0.2 "Crozier"

## Modder's Change Report & Crash-Risk Guide

1.20 "Crozier" was released alongside the **By God Alone** expansion. It is the largest
change to CK3's religion scripting since launch. The Faith is no longer the lowest unit of belief:
every Faith now contains one or more **Rites**. Tenets are no longer doctrines. Laws and law groups are separate
databases, and government "rules" became "mechanics". Almost any mod that touches religion, laws,
governments, portraits, history, or the UI needs to be updated.

This document lists what changed between **1.19.0.6** and **1.20.0.2**, what can crash or silently
break a mod, and how to fix it.

---

## Table of Contents

1. [Summary](#1-summary)
2. [Crash & Breakage Risk Chart](#2-crash--breakage-risk-chart)
   - [High Risk](#-high-risk)
   - [Medium Risk](#-medium-risk)
   - [Low Risk](#-low-risk)
3. [Migration Checklist](#3-migration-checklist)
4. [Religion: The Faith → Rite Restructure](#4-religion-the-faith--rite-restructure)
5. [Faith Keys That Became Rites (lookup table)](#5-faith-keys-that-became-rites-lookup-table)
6. [Tenets Are Now Their Own Database](#6-tenets-are-now-their-own-database)
7. [Doctrines and Doctrine Groups](#7-doctrines-and-doctrine-groups)
8. [History Files](#8-history-files)
9. [Laws and Law Groups](#9-laws-and-law-groups)
10. [Governments](#10-governments)
11. [Spiritual Fulfillment & Personal Tenets](#11-spiritual-fulfillment--personal-tenets)
12. [Holy Orders, Holy Sites, Lease Contracts](#12-holy-orders-holy-sites-lease-contracts)
13. [Puppets](#13-puppets)
14. [Clerical Titles, Clerical Regions & The Christian Church](#14-clerical-titles-clerical-regions--the-christian-church)
15. [On-Actions](#15-on-actions)
16. [Script API: Renamed, Removed, Added](#16-script-api-renamed-removed-added)
17. [Traits, Perks, Men-at-Arms, Artifacts, Titles](#17-traits-perks-men-at-arms-artifacts-titles)
18. [Portraits & Genes](#18-portraits--genes)
19. [GUI & Data Binding](#19-gui--data-binding)
20. [Defines](#20-defines)
21. [Localization](#21-localization)
22. [Map](#22-map)
23. [Other New Modding Features](#23-other-new-modding-features)
24. [Complete Removed-Key Reference](#24-complete-removed-key-reference)
25. [Validating Your Mod](#25-validating-your-mod)
26. [Methodology & Sources](#26-methodology--sources)

---

## 1. Summary

| | |
|---|---|
| **From** | 1.19.0.6 "Scribe" |
| **To** | 1.20.0.2 "Crozier" |
| **Paired DLC** | *By God Alone* — DLC feature flag `by_god_alone`, script prefix `pam_` (checked by `has_pam_dlc_trigger`) |
| **Files added** | 3,228 |
| **Files removed** | 217 |
| **Script/text files changed** | 8,728 (after normalizing line endings and BOMs) |
| **New `common/` folders** | `holy_orders`, `law_groups`, `menu_scenes`, `morpheme_strip_rules`, `puppets`, `spiritual_fulfillment` |
| **New `common/religion/` subfolders** | `faith_types`, `rite_types`, `rite_names`, `rite_icons`, `tenet_types`, `doctrine_category_types` |
| **New `history/` folder** | `history/faiths` (faith history: main Rite, tenet status, tenet popularity by date) |
| **Faith keys that stopped being Faiths** | 62 (58 became Rites, 4 were removed entirely) |
| **Tenets moved out of `doctrine_types`** | 71 |
| **History lines switched from `religion =` to `rite =`** | 76,870 character lines, 8,353 province lines |
| **Removed code on-action** | `on_character_faith_change` (replaced by `on_rite_change`) |
| **Map binaries** | Unchanged (only `geographical_region.txt` changed) |

File counts by top-level folder (not counting `dlc/` or the launcher/root binaries):

| Folder | Added | Removed | Changed |
|---|---:|---:|---:|
| `common/` | 199 | 2 | 1,142 |
| `events/` | 41 | 1 | 487 |
| `gui/` | 44 | 1 | 120 |
| `history/` | 29 | 0 | 443 |
| `localization/` (all languages) | 695 | 88 | 4,092 |
| `gfx/` | 2,216 | 125 | 2,433 |
| `map_data/` | 1 | 0 | 1 |
| `data_binding/` | 2 | 0 | 4 |
| `sound/` | 1 | 0 | 0 |

---

## 2. Crash & Breakage Risk Chart

**How to read this chart**

- 🔴 **High**: Can crash to desktop, stop the game from starting or loading a save, or leave the
  campaign unplayable (characters or counties with no valid faith, realms with no laws, broken portraits).
- 🟠 **Medium**: Usually no crash, but major features silently stop working, error.log floods, or a
  UI window breaks. Some of these can escalate to a crash depending on how the mod uses them.
- 🟢 **Low**: Log errors, missing text or icons, cosmetic problems. The game degrades gracefully.

✔ = **confirmed** in `error.log` by loading a large 1.19-era mod on 1.20.0.2. Everything else is an
assessment based on the file diff, the official patch notes, and known CK3 engine behaviour. It is not a guarantee.

### 🔴 High Risk

| # | What your mod does | What happens in 1.20 | Fix |
|---|---|---|---|
| H1 | **History files** (`history/characters`, `history/provinces`, `history/titles`) set a faith key that is no longer a Faith, e.g. `religion = ashari`, `religion = theravada`, `religion = coptic`, `religion = insular_celtic` | 62 vanilla faith keys are no longer Faiths ([§5](#5-faith-keys-that-became-rites-lookup-table)). Characters or counties can end up with no valid faith at game start. A null faith is a classic CK3 crash cause, and at best it floods the log and breaks conversion, succession, and holy wars. | Use `rite = <key>` for keys that became Rites, or `faith = <new umbrella faith>`. `ashari`, `maturidi`, `mutazila` and `muwalladi` no longer exist anywhere, so use `faith = sunni` plus a madhhab rite (`hanafi`, `maliki`, `shafii`, `hanbali`). |
| H2 | **Custom religions still in 1.19 format** (a `faiths = { ... }` block inside `common/religion/religion_types/*.txt`) | Faith definitions nested inside religions are no longer read. None of those faiths exist, so any history or script that uses them fails as in H1. | Move every faith into `common/religion/faith_types/` as a top-level entry with a `faith_details = { religion = ... }` block ([§4](#4-religion-the-faith--rite-restructure)). |
| H3 | **Full-file overrides of vanilla law files** (`common/laws/00_realm_laws.txt`, `00_succession_laws.txt`, `01_title_succession_laws.txt`, `02_admininistrative_laws.txt`, `03_imperial_policies.txt`) copied from 1.19 | 1.19 laws are nested inside their group. In 1.20 each law must be a top-level entry with `law_group_type`. Group defaults in `common/law_groups` (e.g. `default = crown_authority_1`) point at laws that no longer load, so realms can end up with no valid realm or succession law. This is a strong crash and broken-succession risk. | Convert to flat laws with `law_group_type = <group>` and `index = N`. Move group settings (`default`, `cumulative`, `flag`, `can_change_law_group`) into `common/law_groups/` ([§9](#9-laws-and-law-groups)). |
| H4 | **Full-file override of `common/governments/00_government_types.txt`** copied from 1.19 | The `government_rules = { religious = yes / administrative = yes }` rules are gone. Governments now need `mechanic_type`, with exactly one `is_mechanic_type_default = yes` per mechanic. Vanilla now references `ecclesiastical_government` (Christianity's `theocracy_government_type`) and new law-gating flags. The patch notes list a fixed *"crash from a null government type"*, so null or invalid government types are a known crash path. | Rebuild the override from 1.20 vanilla. Use `mechanic_type`, `is_mechanic_type_default`, and the new flags ([§10](#10-governments)). |
| H5 | **Full-file override of `common/defines/00_defines.txt`** (or other vanilla define files) copied from 1.19 | 44 new defines would be missing: Spiritual Fulfillment min/max, Rite divergence thresholds, holy site caps, rite creation costs, and more. The new systems then read missing or zero values. | Never ship a whole-file copy of vanilla defines. Use a partial define file with only the keys you change ([§20](#20-defines)). |
| H6 | **Full-file overrides of `common/genes/*.txt` or `common/ethnicities/*.txt`** from 1.19 | Genes gained 12 new accessory templates (bibles, croziers, situla, left-hand books), and `eye_accessory` gained a new `no_eyes` entry at index 0, which shifts the index order. Vanilla portrait modifiers and ethnicities reference these, so old overrides can produce broken portraits or portrait crashes and mismatched DNA. | Re-base gene and ethnicity overrides on 1.20 files. Re-export any hard-coded DNA strings ([§18](#18-portraits--genes)). |
| H7 | **Loading a 1.19 save in 1.20** (especially a modded save) | The underlying data model changed: faith → rite, a separate tenet database, law and government restructures. Conversion lookups only exist for a handful of trait and title keys. The patch notes do not say old saves are supported. | Start a new campaign. Tell your users that mid-campaign updates are unsupported. |
| H8 | **Full-file GUI overrides** of any of the **120 changed `.gui` files**, especially `hud.gui`, `hud_bottom.gui`, `window_character.gui`, `window_faith.gui`, `window_title.gui`, `window_decisions.gui`, `window_decisions_detail.gui`, `window_intrigue.gui`, `window_ledger.gui`, `window_activity_planner.gui`. Also overriding or opening the **removed** `window_faith_creation.gui`. | Outdated GUI files call data functions and templates that were removed or renamed (see [§19](#19-gui--data-binding)). Windows fail to build or behave unpredictably, and opening them can crash. | Re-merge your GUI changes onto the 1.20 versions of those files. Faith creation is now **Rite** creation (`window_rite_creation.gui`). |

### 🟠 Medium Risk

| # | What your mod does | What happens in 1.20 | Fix |
|---|---|---|---|
| M1 | ✔ **Defines doctrines without `doctrine_group_type`** | `doctrine 'X' doesn't specify a doctrine_group_type`. The doctrine belongs to no group, cannot be picked, and faiths that list it lose it. | Add `doctrine_group_type = <group>` and `index = N` to every doctrine ([§7](#7-doctrines-and-doctrine-groups)). |
| M2 | **Doctrine groups that list members** (`doctrine_types = { ... }` inside a doctrine group) | Groups no longer own their doctrines; each doctrine names its group instead. The list is ignored or errors. | Delete the `doctrine_types` list from the group and put `doctrine_group_type` on each doctrine. |
| M3 | ✔ **Defines tenets in `common/religion/doctrine_types/`** (1.19 `30_core_tenets.txt` pattern) | They are not tenets anymore. They can't be core or personal tenets, and faith or rite creation ignores them. | Move them to `common/religion/tenet_types/` ([§6](#6-tenets-are-now-their-own-database)). |
| M4 | ✔ **`has_doctrine = tenet_*`, `add_doctrine = tenet_*`, `remove_doctrine = tenet_*`** | `Not found in database class CDoctrineTypeDatabase`. Triggers are always false and effects do nothing. The logic silently inverts (e.g. a "not pacifist" check always passes). | Use `has_tenet = tenet_x` (faith scope) or `rite_has_tenet = tenet_x` (rite scope). For effects, use the tenet effects ([§16](#16-script-api-renamed-removed-added)). |
| M5 | **Old doctrine/tenet parameter syntax** (`parameters = { number_of_spouses = 4 some_flag = yes }`) | Bool parameters are now bare flags. Numeric parameters moved to `special_parameters = { }`. | `parameters = { some_flag }` and `special_parameters = { number_of_spouses = 4 }`. |
| M6 | **Custom or overridden governments without the new law-gating flags** | Realm-law groups now use `required_government_flag`. A government missing the flag silently loses the whole group (no Crown Authority, Tribal Authority, and so on). | Add the needed flag, e.g. `government_uses_crown_authority`, `government_uses_tribal_authority`, `government_uses_church_authority` ([§10](#10-governments)). |
| M7 | **`religion_types` files that put `family`, `graphical_faith`, `piety_icon_group` or `doctrine_background_icon` at the top level** | Religion-level presentation settings now live in `religion_details = { }`. Top-level keys are ignored or error. | Wrap them in `religion_details`. Rename `doctrine_background_icon` to `tenet_background_icon`. |
| M8 | **Hooks on `on_character_faith_change`** | That on-action no longer exists and never fires. | Move the logic to `on_rite_change` (provides `scope:old_faith` and `scope:old_rite`). |
| M9 | **Calls to removed scripted effects or triggers** (`increase_crown_authority_effect`, `create_holy_order_effect`, `trait_is_criminal_in_faith_trigger`, …) | Unknown effect or trigger. The enclosing block fails to validate, so the whole decision, interaction or event option may do nothing. | See the replacement table in [§16](#16-script-api-renamed-removed-added). |
| M10 | **Full-file overrides of vanilla on-action files** (e.g. `religion_on_actions.txt`, anything in `common/on_action/` that changed) | Vanilla wired new Rite, holy site, Church situation and fulfillment logic into these files. Replacing them drops those hooks, and the new systems run in a half-initialized state. | Append to on-actions from your own uniquely named file instead of replacing vanilla files. |
| M11 | ✔ **Lifestyle trees or scripts that reference `scholar_perk`** | `Invalid database object 'scholar_perk'`. The Learning perk tree is broken for your overrides. | Use `erudite_perk`. |
| M12 | **References to `teutonic_knights` MaA, `trinket_4` slot, `d_knights_hospitaler`** | The MaA was replaced by `order_knights` and `order_serjeants`. The `trinket_4` slot was removed (new slots: `brooch`, `holy_relic_1`–`3`). The title was renamed to `d_knights_hospitaller`; a conversion lookup exists for saves and history, but script references should be updated. | Update the keys. |
| M13 | **Full-file override of `common/landed_titles/00_landed_titles.txt`** from 1.19 | Drops `d_knights_hospitaller` and any 1.20 edits. Holy order types and history reference the new key. The new clerical and hegemony titles live in separate `07_pam_*.txt` files and survive. | Re-base on 1.20. Better: use a separate title file. |
| M14 | **Full-file override of `map_data/geographical_regions/geographical_region.txt`** | Drops 70+ new regions (`et_867_*`, `et_1066_*`, `custom_slavic_rite_mission_region`, …) used by clerical-region and Church scripts. | Re-base on 1.20, or add your regions in a separate file. |
| M15 | **Holy sites defined inside a religion's `faiths` block, or faiths with more holy sites than the new caps** | Holy sites are now `holy_sites` and `eminent_holy_sites` on the faith type. Religions can set min/max counts (defaults come from `FAITH_HOLY_SITES_MAX_DEFAULT` and related defines). | Move them to `faith_types`, and check the counts against the caps. |
| M16 | **Faith-creation hooks**: `scripted_rules/faith_creation`, the `faith_creation_piety_cost_add/_mult` modifiers, `faith_creation_cost_mult` script value | All removed. Creation is now Rite creation. | Use `rite_creation_piety_cost_mult` and the `RITE_CREATION_*` defines. |
| M17 | **UI mods that use removed GUI templates or types** (`widget_doctrine_item`, `faith_tooltip_core_tenents`, `container_tenet_item`, `Button_Select_Faith`, `fervor_container_vbox`, `widget_succession_candidate_item`, …) | The widgets fail to instantiate. | See [§19](#19-gui--data-binding). |
| M18 | **Mods that fire removed events** (`fervor.*`, `heresy.0001/0005/0010/0011`, 44 `learn_commander_trait.*` events, `iberia_north_africa.0101/0102/0111/0112`, `east_europe.0015`, `death_management.0098`) | Nothing fires. The fervor event chain is gone and heresy was rebuilt. | Point to the new events or remove the calls. |

### 🟢 Low Risk

| # | What your mod does | What happens in 1.20 | Fix |
|---|---|---|---|
| L1 | Uses `stress_impact = { ... }` | Still valid (vanilla still uses it about 200 times), but it does **not** change Spiritual Fulfillment. | Switch to `stress_and_fulfillment_impact` where piety matters. |
| L2 | ✔ Script references to trait `scholar` | `Failed to find 'scholar' from database for link 'trait'`. Saves and history are auto-converted via `trait_conversion.lookup` (`scholar = erudite`, `eunuch = eunuch_1`, `poet = lifestyle_poet`), but script is not. | Use `erudite`, `eunuch_1`, `lifestyle_poet`. |
| L3 | Uses removed modifiers: `ashari_opinion`, `imami_opinion`, `maturidi_opinion`, `mutazila_opinion`, `muwalladi_opinion`, `faith_creation_piety_cost_add/_mult`, `prepare_travels_modifier`, `rejected_from_marriage_bed_modifier` | Unknown-modifier errors. The modifier does nothing. | Remove or replace them. |
| L4 | Sets the removed defines `CONVERGENCE_DELAY`, `FAITH_CREATION_FERVOR_DISCOUNT_MAX`, `FAITH_CREATION_FERVOR_DISCOUNT_PER_MISSING_FERVOR`, `MINIMUM_FAITH_SIZE_FERVOR_MODIFIER` | Log error only. | Remove them; the replacements are the `RITE_*` defines. |
| L5 | Relies on one of 616 removed English localization keys | The raw key shows in game. | Re-point to new keys. |
| L6 | Uses `doctrine_background_icon` or icons in the old doctrine background folder | Missing texture. | Use `tenet_background_icon`; the textures moved to the `faith_tenets` folder. |
| L7 | Uses `character_faith_modifier` on holding buildings | No longer supported. | Use another modifier type. |
| L8 | Uses `lease_out_to` | Renamed. | `lease_out_to_holy_order`. |
| L9 | Uses `has_commander_trait_trigger` | Removed. | `has_trait_with_flag`. |
| L10 | Uses `house ?= { has_base_name = ... }` in CoA template lists | Vanilla switched to `dynasty_has_base_name`, which is new. | Prefer the new trigger where you mean the dynasty. |
| L11 | Overrides the removed decisions `ai_create_head_of_faith_decision`, `appoint_a_righteous_caliph_decision`, `mozarabic_bind_the_faith_to_rome_decision`, `mozarabic_break_with_rome_decision` | Orphaned override. | Remove it. |
| L12 | Your `.mod` descriptor says `supported_version="1.19.*"` | The launcher shows an "out of date" warning. This does not crash anything. | Set `supported_version="1.20.*"` once updated. |
| L13 | Map mods (provinces, heightmap, rivers, `definition.csv`, `default.map`) | Map binaries are byte-identical between 1.19.0.6 and 1.20.0.2. | Nothing to do, apart from `geographical_region.txt` (M14). |

---

## 3. Migration Checklist

Work top to bottom; the early items are the ones that crash.

- [ ] Search every file you ship for `faiths = {` inside `religion_types`, and move each faith to `common/religion/faith_types/`.
- [ ] Search history for `religion =` and `faith =`, and map every removed faith key with [§5](#5-faith-keys-that-became-rites-lookup-table). Prefer `rite = <key>` plus `faith = <umbrella>`.
- [ ] Convert every file in `common/laws/` to flat laws (`law_group_type`, `index`), and move group settings to `common/law_groups/`.
- [ ] Rebuild government overrides with `mechanic_type`, `is_mechanic_type_default`, and the new flags.
- [ ] Delete whole-file copies of vanilla defines, genes, ethnicities, GUI, on-actions and landed titles. Re-base any you must keep on 1.20.
- [ ] Add `doctrine_group_type`, `index` and `divergence` to every doctrine, and remove `doctrine_types = {}` lists from groups.
- [ ] Move every `tenet_*` definition to `common/religion/tenet_types/`.
- [ ] Replace `has_doctrine = tenet_*` with `has_tenet` or `rite_has_tenet`.
- [ ] Convert doctrine and tenet `parameters` to bare flags plus `special_parameters`.
- [ ] Replace `on_character_faith_change` with `on_rite_change`.
- [ ] Replace removed scripted effects and triggers ([§16](#16-script-api-renamed-removed-added)).
- [ ] Replace `scholar`/`scholar_perk`, `teutonic_knights`, `trinket_4`, `d_knights_hospitaler`.
- [ ] Start a **new** game with `-debug_mode`, then read `Documents/Paradox Interactive/Crusader Kings III/logs/error.log` ([§25](#25-validating-your-mod)).
- [ ] Bump `supported_version` in your `.mod` descriptor.

---

## 4. Religion: The Faith → Rite Restructure

### New hierarchy

```
religion_family_type   (unchanged)   e.g. rf_abrahamic
└─ religion_type       (slimmed)     e.g. christianity_religion
   └─ faith_type       (NEW file)    e.g. christian_faith, catholic, orthodox, miaphysitism
      └─ rite_type     (NEW)         e.g. roman_rite, byzantine_rite, coptic_rite, insular_celtic
```

- A **Faith** now owns the Head of Faith, holy sites, cultures, reserved names, military order names,
  and a `main_rite`.
- A **Rite** owns core tenets, rite-specific doctrines, color, icon, founder title, and its own **Head of Rite**.
  Characters and counties hold a Rite, and their Faith is the Rite's parent.
- A faith without a scripted Rite gets a dynamic Rite with the same key, which is why keys like
  `norse_pagan` or `akom_pagan` still work in history.
- Rites can split off, diverge (**divergence** is a new concept), become heretical, and change parent Faith.
  Doctrines can be locked at Rite, Faith or Religion level and are lost when a Rite moves past that level.

### `religion_types` — 1.19 vs 1.20

1.19 (everything in one file):

```
christianity_religion = {
    family = rf_abrahamic
    graphical_faith = catholic_gfx
    doctrine = ...
    doctrine_selection_pair = { ... }
    faiths = {
        catholic = { color = { ... } doctrine = ... holy_site = ... }
    }
}
```

1.20 (`00_christianity.txt` went from 1,154 to 336 lines):

```
christianity_religion = {
    religion_details = {
        family = rf_abrahamic
        piety_icon_group = "christian"
        theocracy_government_type = ecclesiastical_government
        theocracy_lease_contract_type = ecclesiastical_lease
    }
    main_holy_site = key            # optional; cannot be removed from any faith of the religion
    eminent_holy_sites_max = 3      # optional caps; defaults come from defines
    holy_sites_max = 9
    eminent_holy_sites_min = 0
    holy_sites_min = 1
    doctrine = ...                  # religion-wide doctrines stay here
    traits = { virtues = { ... } sins = { ... } }   # weights now feed Spiritual Fulfillment
}
```

Removed from religions: the `faiths = { }` block, `doctrine_selection_pair`, top-level `family`,
`graphical_faith`, `piety_icon_group` and `doctrine_background_icon` (now `tenet_background_icon` inside `religion_details`).

### `faith_types` (new — `common/religion/faith_types/`)

```
catholic = {
    main_rite = roman_rite
    faith_details = {
        religion = christianity_religion      # required
        color = { 0.8 0.8 0.6 }
        icon = ...
        reformed_icon = ...
        religious_head = title_key
        head_of_rite = title_key               # overrides who leads the main Rite
        graphical_faith = "catholic_gfx"
        theocracy_government_type = government_type_key
    }
    origin = christian_faith                   # history can accept HoFs from the origin faith without errors
    clerical_elector_titles = { d_cd_ostia ... }
    holy_sites = { ... }                       # non-eminent: local bonuses only
    eminent_holy_sites = { ... }               # also give global bonuses; never list a site twice
    tenets = { ... }                           # only used to seed a dynamic main Rite
    doctrines = { ... }                        # additive to the main Rite; Rite doctrines win per group
    tenet_selection_pair = { requires_dlc_flag = ... tenet = ... fallback_tenet = ... }
    reserved_male_names = { ... }
    reserved_female_names = { ... }
    cultures = { ... }
    historical = no
    localization = { ... }
    holy_order_names = { ... }
}
```

### `rite_types` (new — `common/religion/rite_types/`, 152 vanilla rites)

```
roman_rite = {
    name = roman_rite
    desc = roman_rite_desc
    founder = k_papal_state        # first holder of this title becomes the founder / Head of Rite
    faith = christian_faith        # parent faith; no parent = only created via create_rite_from_type
    color = { 0.8 0.8 0.6 }
    icon = christianity_papal_cross_01
    cultures = { italian roman lombard }
    convert = yes                  # can characters convert to it
    create = yes                   # no = not created until history says so; ALWAYS check validity in script
    tenets = { tenet_communion }   # the faith's effective core tenets if this is the main Rite
    tenet_selection_pair = { requires_dlc_flag = by_god_alone tenet = tenet_dulia fallback_tenet = tenet_armed_pilgrimages }
    doctrines = { ... }            # override religion/faith doctrines in the same group
}
```

Related new databases: `rite_names`, `rite_icons`, `doctrine_category_types`, and dynamic holy sites
in `holy_site_types/01_dynamic_holy_site_types.txt`.

> **Warning from the `.info`**: rites with `create = no` error when accessed via `rite:key`. Always
> check `exists = rite:key` in script.

### Faiths per religion (1.20 vanilla)

| Religion | Faiths |
|---|---|
| `christianity_religion` | `christian_faith` (pre-schism umbrella), `catholic`, `orthodox`, `miaphysitism`, `armenian_apostolic`, `conversos`, `cathar`, `waldensian`, `lollard`, `hussite`, `joachimite`, `iconoclast`, `bogomilist`, `paulician`, `nestorian`, `messalian`, `adamites`, `bosnian_church`, `adoptionist` |
| `islam_religion` | `sadr_al_islam`, `sunni`, `shia`, `kharijite`, `masmudi`, `quranist`, `druze`, `qarmatian` |
| `judaism_religion` | `rabbinic_faith`, `karaism`, `haymanot`, `malabarism`, `samaritan`, `kabarism` |
| `buddhism_religion` | `theravada_faith`, `mahayana_faith`, `maitreya_faith`, `vajrayana_faith` |
| `hinduism_religion` | `vaishnava_faith`, `shaiva_faith`, `shakta_faith`, `smartism`, `saura` |

Christian rites under `christian_faith`: `roman_rite`, `insular_celtic`, `mozarabic_church`,
`ambrosian_rite`, `beneventan_rite`, `carolingian_christianity`, `byzantine_rite`, `georgian_rite`,
`slavic_rite`. `catholic` has `main_rite = roman_rite` and `orthodox` has `main_rite = byzantine_rite`;
faith history moves the rites between faiths at the Great Schism.
Under `miaphysitism`: `coptic_rite`, `antiochian_rite`, `ethiopian_rite`.
Under `sunni`: `hanafi` (main), `maliki`, `shafii`, `hanbali`, `sahabah`, `tabiun`.
Under `shia`: `ismaili` (main), `hafizi`, `alawite`, `alevi`, `nizari`, `zayidi`, `imami`, `ghulat`.
Under `kharijite`: `ibadi`, `azariqa`, `najdat`, `sufri`.

---

## 5. Faith Keys That Became Rites (lookup table)

These keys were **Faiths in 1.19** and are **not Faiths in 1.20**. `faith:<key>` scopes,
`religion = <key>` / `faith = <key>` in history, and `set_character_faith = faith:<key>` will all fail.
In vanilla 1.19 the most-referenced were `faith:mozarabic_church` (66), `faith:theravada` (59), `faith:ashari` (58), `faith:coptic` (49), `faith:manichean` (47), `faith:insular_celtic` (44), `faith:ismaili` (40).

| Old 1.19 faith key(s) | 1.20: Rite key(s) | 1.20 parent Faith |
|---|---|---|
| `insular_celtic`, `mozarabic_church` | same keys | `christian_faith` |
| `coptic` | **`coptic_rite`** (renamed) | `miaphysitism` |
| `alawite`, `alevi`, `ghulat`, `hafizi`, `imami`, `ismaili`, `nizari`, `zayidi` | same keys | `shia` |
| `azariqa`, `ibadi`, `najdat`, `sufri` | same keys | `kharijite` |
| `ashari`, `maturidi`, `mutazila`, `muwalladi` | **removed entirely** | use `sunni` + `hanafi` / `maliki` / `shafii` / `hanbali` |
| `merkabah`, `rabbinism` | same keys | `rabbinic_faith` |
| `cainitism`, `priscillianism`, `valentinianism` | same keys | `gnostic_faith` |
| `mandeaism`, `sabianism` | same keys | `mandaean_faith` |
| `manichean`, `mingism` | same keys | `manichaean_faith` |
| `kitebacilweism`, `meshefaresism`, `yazidi` | same keys | `yazidi_faith` |
| `theravada` | same key | `theravada_faith` |
| `avatamsaka`, `dhyana`, `mahayana`, `pundarika`, `sukhavati`, `vinaya`, `yogacara` | same keys | `mahayana_faith` |
| `maitreya` | same key | `maitreya_faith` |
| `acharya`, `ari`, `lamaism`, `mantrayana`, `vajrayana` | same keys | `vajrayana_faith` |
| `advaitism`, `shaivism` | same keys | `shaiva_faith` |
| `kalikula_shaktism`, `srikula_shaktism` | same keys | `shakta_faith` |
| `krishnaism`, `vaishnavism` | same keys | `vaishnava_faith` |
| `digambara`, `svetambara`, `yapaniya` | same keys | `jain_faith` |
| `bon`, `old_bon` | same keys | `bon_faith` |
| `daoxue`, `jingxue` | same keys | `confucian_faith` |
| `quanzhen`, `shangqing` | same keys | `shangqing_faith` |
| `shinto`, `shugendo` | same keys | `shinto_faith` |

**New Faith keys in 1.20:** `bon_faith`, `christian_faith`, `confucian_faith`, `gnostic_faith`,
`hussite`, `jain_faith`, `joachimite`, `kharijite`, `mahayana_faith`, `maitreya_faith`,
`mandaean_faith`, `manichaean_faith`, `miaphysitism`, `rabbinic_faith`, `sadr_al_islam`, `shaiva_faith`,
`shakta_faith`, `shangqing_faith`, `shia`, `shinto_faith`, `sunni`, `theravada_faith`,
`vaishnava_faith`, `vajrayana_faith`, `yazidi_faith`.

**Rules of thumb for scripts**
- "Is this character Imami?" → `rite = rite:imami` (or `rite_has_*` triggers), **not** `faith = faith:imami`.
- "Is this character Shia?" → `faith = faith:shia`.
- Faiths are 103 vanilla keys and rites are 152. 124 old faith keys are now also Rite keys, because every
  faith without a scripted Rite gets a same-named dynamic one.

---

## 6. Tenets Are Now Their Own Database

- **Moved:** 71 `tenet_*` entries from `common/religion/doctrine_types/30_core_tenets.txt` (file
  deleted) to `common/religion/tenet_types/00_tenet_types.txt`. There is also `00_pam_tenets.txt` for DLC tenets.
- **Removed:** the `doctrine_core_tenets` doctrine group.
- **Keys kept their names** (`tenet_pacifism`, `tenet_communion`, …), which is why broken
  `has_doctrine = tenet_x` calls look correct but always fail.
- New tenet fields: `divergence_multiplier`, `can_pick_as_personal_tenet`, `personal_tenet_modifier`,
  `personal_tenet_parameters`, `special_parameters`, `requires_dlc_flag`, and a `piety_cost` scoped to the **Rite**.
- Tenets have a **status** per faith (known / permitted / prohibited) and a **popularity**,
  both seeded by `history/faiths`.
- Cap: `FAITH_CORE_TENETS_CAP` and `BASE_PERSONAL_TENETS_CAP` defines.

| 1.19 | 1.20 |
|---|---|
| `has_doctrine = tenet_x` (faith scope) | `has_tenet = tenet_x` (faith scope) |
| n/a | `rite_has_tenet = tenet_x` (rite scope) |
| n/a | `has_personal_tenet = tenet_x`, `has_personal_tenet_flag = flag` (character) |
| n/a | `knows_tenet`, `has_tenet_status`, `has_tenet_popularity_trigger` |
| n/a | `add_known_tenet`, `add_tenet`, `change_tenet_popularity`, `set_order_tenet` |
| n/a | lists: `any_/every_/random_/ordered_personal_tenet`, `any_character_tenet`, `any_desired_tenet` |
| `doctrine_selection_pair` / `fallback_doctrine` | `tenet_selection_pair` / `fallback_tenet` |
| `scope:` n/a | `tenet:tenet_key` scope |

---

## 7. Doctrines and Doctrine Groups

Doctrines now **declare their own group**, just as laws now do. Before, the group listed its doctrines.

1.19:

```
# doctrine_group_types
doctrine_marriage_type = {
    category = "marriage"
    doctrine_types = { doctrine_monogamy doctrine_polygamy doctrine_concubines }
}
# doctrine_types
doctrine_monogamy = {
    piety_cost = { ... has_doctrine = doctrine_monogamy ... }
    parameters = { number_of_spouses = 1 marriage_event = yes }
}
```

1.20:

```
# doctrine_group_types
doctrine_marriage_type = {
    category = marriage             # now a key into doctrine_category_types
    divergence = sequence           # category | sequence
    doctrine_lock = ...             # optional: rite / faith / religion lock
}
# doctrine_types
doctrine_monogamy = {
    doctrine_group_type = doctrine_marriage_type   # REQUIRED
    index = 0                                      # order within group; also used for divergence
    divergence = 0
    piety_cost = { ... rite_has_doctrine = doctrine_monogamy ... }   # cost is evaluated in RITE scope
    parameters = { doctrine_monogamy marriage_event }                # bool params are bare flags
    special_parameters = { number_of_spouses = 1 }                   # non-bool params live here
}
```

Other doctrine changes:
- New database `common/religion/doctrine_category_types/` (categories used to be free strings).
- New `DescriptiveName` localization support for doctrines (e.g. "Criminal Deviancy").
- New `change_doctrine` effect, which replaces the existing doctrine of the same group.
- New `rite_has_doctrine`, `rite_has_parameter`, `rite_can_choose_doctrine`, `knows_doctrine`, `add_known_doctrine`.
- Faith doctrines are **additive** to the main Rite. Rite doctrines win within the same group.
- `special_parameters` gains `divergence_protection`.

---

## 8. History Files

| File type | 1.19 | 1.20 |
|---|---|---|
| `history/characters/*.txt` | `religion = "waaqism_pagan"` | `rite = "waaqism_pagan"` (`faith = x` gives the faith's main Rite; `religion = <faith>` is kept as a compatibility alias **for faith keys only**) |
| `history/provinces/*.txt` | `religion = mantrayana` | `rite = mantrayana`; define both `faith` and `rite`. The game falls back to the faith's main Rite if the Rite doesn't exist at that bookmark |
| `history/faiths/*.txt` | n/a | **New.** File name = faith key. Dated blocks set `created`, `main_rite`, `religious_head`, `known`/`permitted`/`prohibited` tenets, `popularity`, and rite descriptions |
| `history/titles/ce3/00_ecclesiastical_titles.txt` | n/a | New holders for clerical titles |
| `history/characters/ecclesiastical.txt`, `saints.txt` | n/a | New historical clergy and saints |
| `history/situations/pam_the_christian_church_history.txt` | n/a | New Christian Church situation history |
| `history/cultures/` | — | +8: `bouxcuengh`, `jurchen`, `kachin`, `khitan`, `ongud`, `shatuo`, `tuyuhun`, `yughur` |

New history features:
- `history_override_priority = N` overrides a single vanilla character from your own file, with no whole-file override needed.
- `effect_even_if_dead` for character history.
- All historical characters are now permanently unprunable.
- "Obscured" historical characters (unclickable) with a `<key>_obscured_desc` tooltip.
- `destroy_landless_title_no_tgp_dlc_effect` / `_no_dlc_effect` are no longer used in `history/titles` (removed as scripted effects).

---

## 9. Laws and Law Groups

**The biggest non-religion breaking change.** Groups and laws are now two databases, so adding a law
no longer means editing (overriding) the group file. **All law keys and group keys are unchanged.** Only the structure moved.

1.19 (`common/laws/00_realm_laws.txt`):

```
crown_authority = {
    default = crown_authority_1
    cumulative = yes
    flag = realm_law
    crown_authority_0 = { modifier = { ... } can_keep = { ... } }
    crown_authority_1 = { ... }
}
```

1.20 (`common/law_groups/00_realm_law_groups.txt` + `common/laws/00_realm_laws.txt`):

```
# law_groups
crown_authority = {
    required_government_flag = { government_uses_crown_authority }   # fast-path gate
    default = crown_authority_1
    cumulative = yes
    flag = realm_law
}
# laws
crown_authority_0 = {
    law_group_type = crown_authority      # REQUIRED
    index = 0                             # order in group
    modifier = { ... }
    can_keep = { ... }
}
```

| New law-group field | Purpose |
|---|---|
| `required_government_flag = { ... }` | The government must have one of these flags, or the group is invalid |
| `can_have_group = { }` | Trigger: can this ruler have any law of this group |
| `is_treasury_budget_group` | Shown in the Treasury Budget UI |
| `can_change_law_group` | Still supported (visible but locked) |

New vanilla groups include `church_authority`, `budget_allocation_cardinal_law` and
`church_benefices_law` (both need `government_is_ecclesiastical`), plus `04_pam_ecclesiastical_law_groups.txt`.

Scripted-effect consolidation: `increase_/decrease_crown_authority_effect`,
`increase_/decrease_tribal_authority_effect`, `increase_/decrease_nomadic_authority_effect` and
`increase_/decrease_imperial_bureaucracy_effect` were **removed**. The single **`change_authority_effect`** now handles every authority step.

Also: the law flag is checkable in script with `has_realm_law_flag`, and hot-reloading cumulative laws no longer duplicates effects.

---

## 10. Governments

| 1.19 | 1.20 |
|---|---|
| `government_rules = { religious = yes }` | `mechanic_type = theocracy` |
| `government_rules = { administrative = yes }` | `mechanic_type = administrative` (requires the `admin_gov` DLC flag) |
| n/a | `is_mechanic_type_default = yes`: exactly **one** per mechanic type, used when spawning characters or changing government |
| n/a | Trigger `government_has_mechanic`; `government_type_has_flag` for the government-type scope |
| `royal_court = none / any / top_liege` | adds `landed` |
| n/a | `possible_grant_vassal_governments = { ... }` + `grant_vassal_ai_will_do`: government picker in Grant Titles |
| n/a | `redirects_wars_to_overlord`, `treasury_vassal_development`, `add_religious_subordinates_for_treasury` |

Supported `mechanic_type` values: `feudal`, `mercenary`, `holy_order`, `clan`, `theocracy`,
`administrative`, `landless_adventurer`, `herder`, `nomad`, `mandala`.

**New file:** `common/governments/02_theocratic_government_types.txt` (`ecclesiastical_government`).
Religions and faiths pick their theocracy via `theocracy_government_type`, and their lease via `theocracy_lease_contract_type`.

**Flags that law groups now require.** If your custom government should keep a law group, it needs the flag:

| Law group | Required flag | Present in 1.19 vanilla governments? |
|---|---|---|
| `crown_authority` | `government_uses_crown_authority` | yes |
| `tribal_authority` | `government_uses_tribal_authority` | **no — new** |
| `church_authority` | `government_uses_church_authority` | **no — new** |
| `imperial_bureaucracy` | `government_uses_imperial_bureaucracy` | **no — new** |
| `celestial_bureaucracy`, `grand_secretariat_laws`, `candidate_score_laws` | `government_uses_celestial_bureaucracy` | **no — new** |
| `japanese_bureaucracy` | `government_uses_japanese_bureaucracy` | **no — new** |
| `meritocratic_bureaucracy` | `government_uses_meritocratic_bureaucracy` | **no — new** |
| `budget_allocation_cardinal_law`, `church_benefices_law` | `government_is_ecclesiastical` | **no — new** |
| `budget_allocation_*_law` | `government_has_treasury` | yes |
| `camp_purpose` | `government_is_landless_adventurer` | yes |
| `nomadic_authority` | `government_is_nomadic` | yes |
| `mandala_decree` | `government_is_mandala` | yes |
| `imperial_policy_laws` | `government_is_japan_administrative` | yes |

Admin scripting was rewritten to use explicit mechanics and flags, so mods can build non-vanilla admin
governments (e.g. an admin realm without noble families).

---

## 11. Spiritual Fulfillment & Personal Tenets

- **Spiritual Fulfillment** is a new character stat, defined per religion in `common/spiritual_fulfillment/`
  (levels with thresholds, modifiers, icons, and flags checked by `has_fulfillment_parameter`).
  Exactly one type may have an empty `religions` list as the fallback, and types must not overlap.
- Virtue and sin trait `weight` in religions now **is** base fulfillment. It is multiplied per source by
  `SPIRITUAL_FULFILLMENT_MULT_RELIGION_TRAIT / _CORE_TENET / _PERMITTED_TENET / _PERSONAL_TENET`.
- **`stress_and_fulfillment_impact`** replaced `stress_impact` in vanilla (12,102 → about 200 remaining uses).
  It also moves fulfillment based on the traits and the character's Rite; only negative-stress, "compatible" traits count.
- Effects and triggers: `change_spiritual_fulfillment`, `has_spiritual_fulfillment_type`,
  `character_spiritual_fulfillment_gain_mult`, `rite_fulfillment_score`.
- **Personal Tenets:** characters can hold tenets that differ from their faith (`BASE_PERSONAL_TENETS_CAP`,
  `PERSONAL_TENET_CHANGE_COOLDOWN_DAYS`). Hooks: `on_personal_tenet_gain`, `on_personal_tenet_loss`.
- AI characters try to create Rites that maximize their Spiritual Fulfillment.
- Defines: `MIN_SPIRITUAL_FULFILLMENT`, `MAX_SPIRITUAL_FULFILLMENT`, `MIN_FULFILLMENT_CHANGE_FOR_EFFECT`,
  `SPIRITUAL_FULFILLMENT_FOR_STRESS_DIVIDE_FACTOR`, `MAX_VIRTUES_SINS`.

---

## 12. Holy Orders, Holy Sites, Lease Contracts

**Holy Orders** (`common/holy_orders/`, new):
- Data-driven, weighted types: `type = military | monastic | mendicant`, locked `government`, `trigger`,
  `weight`, and an optional fixed `title` (e.g. `d_knights_templar`) and `name`.
- Vanilla files: `00_christian_holy_orders.txt`, `00_muslim_holy_orders.txt`, `00_generic_holy_orders.txt`.
- The `create_holy_order_effect` scripted effect was removed. Use the `create_holy_order` effect and
  `create_holy_order_neutral_effect` / `create_holy_order_accompanying_effect`.
- `lease_out_to` → `lease_out_to_holy_order`.
- Men-at-arms: `teutonic_knights` removed; `order_knights` and `order_serjeants` added.
- Religion `holy_order_names` stay as a fallback, but vanilla now prefers Holy Order Types for flavour.

**Holy Sites:**
- Faiths have `holy_sites` (local) and `eminent_holy_sites` (global bonuses).
- Religions have min/max caps, and defines `FAITH_HOLY_SITES_MAX_DEFAULT`, `FAITH_EMINENT_HOLY_SITES_MAX_DEFAULT`, etc.
- Dynamic holy sites (`01_dynamic_holy_site_types.txt`, `CREATE_HOLY_SITE_RARITY_REQUIRED`, `MAX_HOLY_SITE_RARITY_TOTAL`).
- New `holy_site:` scope. Artifacts can be enshrined at holy sites (slot `holy_site = yes`, `available_holy_site` trigger, `ARTIFACT_DURABILITY_DECAY_ENSHRINED`).
- On-actions: `on_faith_holy_site_added/_removed/_eminence_changed`, `on_character_holy_site_created`,
  `on_holy_site_artifact_enshrined/_removed`, `on_artifact_changed_holy_site_owner`.
- Removed effect localization: `activate_holy_site`, `deactivate_holy_site`.

**Lease contracts:** configurable per faith and religion (`theocracy_lease_contract_type`); vanilla
adds `common/lease_contracts/00_theocracy_lease.txt`.

---

## 13. Puppets

New database, `common/puppets/types/` and `common/puppets/actions/`. Rulers act through proxy characters.

- `puppet_types`: `interaction` (you must call `set_puppet` yourself), `can_have`, `is_valid` (checked
  daily), `priority`, `actions`, and triggered `asset` / `animation` for the UI.
- `puppet_actions`: `type = character_interaction | decision | great_project`, `is_enabled`, `is_important`.
  In **decision** puppet actions, **ROOT is the puppet** and `scope:puppeteer` is the ruler.
- New interaction scope `puppet_or_actor`: the "real" initiator, while `actor` stays the puppeteer.
- New `puppet:` scope. Vanilla example: `theological_agent_puppet`, `external_theological_agent_puppet`.

---

## 14. Clerical Titles, Clerical Regions & The Christian Church

- **About 290 clerical titles** (`d_cd_*` cardinal sees, dioceses) in `common/landed_titles/07_pam_ecclesiastical_titles.txt`.
- **Hegemony title** `h_kingdom_of_heaven` in `07_pam_hegemony_titles.txt`.
- **Clerical regions**: effects to create, merge and split them, and to mark clerical electors; triggers for
  region size and adjacency. `clerical_region`, `clerical_region_title`, `any_/every_county_in_clerical_region`, `head_of_rite`.
  Defines: `MONTHLY_RITE_CONVERSION_PROCESS_PER_NEIGHBORING_COUNTY`, `REQUIRE_SAME_CHAPLAIN_RITE_FOR_REGION_CONVERSION`.
- **Succession:** `common/succession_election/00_pam_clerical_elective.txt` (papal elections),
  `common/succession_appointment/clerical_christian.txt`. Election candidates now carry a list key, and
  `allow_same_candidate_tier` is now an enum (was bool).
- **Situation:** `the_christian_church` (`common/situation/situations/pam_christian_situation.txt`) with catalysts, Great Schism progress, Papal Bulls.
- **Activities:** `ecumenical_council`, `pam_investiture_council`.
- **Schemes:** `study_faith`, `pam_study_scheme` (study scripture), `find_follower_of_faith`.
- **Other:** Antipopes, saints (`add_saint`, `has_saint_character`, relic creation), Excommunication rework,
  a Grand Cathedral great project, an ecclesiastical domicile (244 new domicile building keys), 867 bookmark Christianity content, and Empire Faith Gate game rules.
- Removed scripted GUIs `create_head_of_faith` / `recreate_head_of_faith`, and related effects and triggers
  (`create_head_of_faith_title_effect`, `force_create_head_of_faith_title_Effect`, `can_afford_create_head_of_faith_title_cost_trigger`).

---

## 15. On-Actions

**Removed:** `on_character_faith_change` (code on-action) and `faith_fervor_events_pulse`.
**Moved, still present:** `on_government_change` moved from `dlc/mpo/mpo_on_actions_2.txt` to `government_on_actions.txt`.

**New code on-actions:**

| On-action | Root / scopes |
|---|---|
| `on_rite_change` | character; `scope:old_faith`, `scope:old_rite` (may equal the current rite). Fires on faith **or** rite change, not at birth or creation |
| `on_county_rite_change` | county title; `scope:old_rite`. Not fired when the county converts to a different faith |
| `on_rite_created`, `on_rite_edited`, `on_rite_updated`, `on_rite_yearly`, `update_main_rite_action` | rite |
| `on_character_created` | fires whenever script or code creates a character (not births); documented creation reasons |
| `on_trait_gained`, `on_trait_lost` | character; `scope:trait` |
| `on_player_character_change` | new player character; `scope:previous_player_character` |
| `on_personal_tenet_gain`, `on_personal_tenet_loss` | character |
| `on_faith_holy_site_added`, `on_faith_holy_site_removed`, `on_faith_holy_site_eminence_changed` | faith / holy site |
| `on_character_holy_site_created`, `on_holy_site_artifact_enshrined`, `on_holy_site_artifact_removed`, `on_artifact_changed_holy_site_owner` | holy sites & artifacts |
| `on_clerical_region_faith_mismatch` | clerical region |
| `on_challenger_removed` | religious head challenger |
| `on_liberation_siege_completion` | siege |

Also: the reason for death is now passed directly to `on_death`; points of interest fire `on_add` and `on_remove`;
`on_house_relation_level_changed` now fires reliably. There are about 20 new `pam_*`, `ecumenical_*` and pulse on-actions.

---

## 16. Script API: Renamed, Removed, Added

### Renamed / replaced

| 1.19 | 1.20 |
|---|---|
| `has_doctrine = tenet_x` | `has_tenet = tenet_x` / `rite_has_tenet` |
| `has_doctrine = doctrine_x` (for rite-specific data) | still valid in faith scope; `rite_has_doctrine` in rite scope |
| `stress_impact` | `stress_and_fulfillment_impact` (old still valid) |
| `doctrine_character_modifier` | `faith_character_modifier` (+ `faith_character_modifier_scaled`) |
| `involved_doctrine_character_modifier` | `involved_faith_character_modifier` |
| `interloper_doctrine_character_modifier` | `interloper_faith_character_modifier` |
| `doctrine_selection_pair` / `fallback_doctrine` | `tenet_selection_pair` / `fallback_tenet` |
| `doctrine_background_icon` | `tenet_background_icon` |
| `set_state_faith` | `set_state_rite` (+ `state_rite`) |
| `lease_out_to` | `lease_out_to_holy_order` |
| `has_commander_trait_trigger` | `has_trait_with_flag` |
| `increase_/decrease_<x>_authority_effect` | `change_authority_effect` |
| `create_holy_order_effect` | `create_holy_order` / `create_holy_order_neutral_effect` |
| `trait_is_criminal_in_faith_trigger` | `trait_is_criminal_in_rite_trigger` |
| `trait_is_shunned_in_faith_trigger` | `trait_is_shunned_in_rite_trigger` |
| `trait_is_shunned_or_criminal_in_faith_trigger` | `trait_is_shunned_or_criminal_in_rite_trigger` |
| `trait_is_shunned_or_criminal_in_my_or_lieges_faith_trigger` | `trait_is_shunned_or_criminal_in_my_or_lieges_rite_trigger` |
| `relation_with_character_is_incestuous_in_faith_trigger` | `relation_with_character_is_incestuous_in_rite_trigger` |
| `relation_with_character_is_incestuous_in_my_or_lieges_faith_trigger` | `relation_with_character_is_incestuous_in_my_or_lieges_rite_trigger` |
| `relation_with_character_is_sodomy_in_my_or_lieges_faith_trigger` | `relation_with_character_is_sodomy_in_my_or_lieges_rite_trigger` |
| (trait virtue/sin checks) | `trait_is_virtue_rite`, `trait_is_sin_rite` |
| `guardian_or_court_tutor_trait` / `guardian_or_court_tutor_trigger_event` | `guardian_or_court_tutor_trait_trigger` / `guardian_or_court_tutor_trigger_event_effect` |
| `house ?= { has_base_name = x }` (CoA lists) | `dynasty_has_base_name = x` |
| `faith_creation_piety_cost_mult` | `rite_creation_piety_cost_mult` |
| `realm_titles` interaction target (old behavior) | `realm_counties` (`realm_titles` now adds **all** titles) |

### Added (highlights)

- **Scopes:** `rite:`, `tenet:`, `puppet:`, `holy_site:`, `situation:the_christian_church`.
- **Rite:** `set_character_rite`, `set_character_rite_with_conversion`, `set_county_rite`, `create_rite_from_type`,
  rite divergence effects, `set_faith_name`, `set_rite_name`, `rite_religion_tag`, `rite_hostility_level`,
  `rite_modifier`, `rite_counties`, `head_of_rite` / `head_of_rites` interaction targets, and a `rite_conversion` special interaction.
- **Tenets:** see [§6](#6-tenets-are-now-their-own-database).
- **Doctrine:** `change_doctrine`, `rite_has_doctrine`, `rite_has_parameter`, `rite_can_choose_doctrine`, `knows_doctrine`, `add_known_doctrine`.
- **Government:** `government_has_mechanic`, `government_type_has_flag`, `is_landed_or_landless_administrative_or_theocratic`.
- **Characters:** `grandchild`, `great_grandchild`, `real_parent`, `real_grandparent`, `real_great_grandparent` lists;
  `clear_father`, `clear_mother`, `clear_real_father`, `clear_real_mother`; birth-date triggers `days_since_birth`,
  `birth_date`, `day_of_year_of_birth`, `month_of_year_of_birth`, `day_of_month_of_birth`; `change_legitimacy_level`.
- **Misc:** `expose_scheme_to`, `valid_for_council_task`, `valid_for_council_position`, `has_court_position` with a scope,
  `create_domicile_title`, a retire-accolade scripted effect, `other_faith_heads` ai_recipients,
  `?=` optional checks inside script values, and `can_create_title` / `can_destroy_title` scripted rules.
- **Standalone currency modifiers** (lump sums only): `prestige_standalone_gain_mult`, `piety_standalone_gain_mult`, plus Influence and Merit versions.
- **Traits:** the `inheritance_blocker` field (limits inheritance to secular titles only).

### Removed scripted effects (26)

`create_head_of_faith_title_effect`, `create_holy_order_effect`, `decrease_crown_authority_effect`,
`decrease_imperial_bureaucracy_effect`, `decrease_nomadic_authority_effect`, `decrease_tribal_authority_effect`,
`destroy_landless_title_no_dlc_effect`, `destroy_landless_title_no_tgp_dlc_effect`,
`expand_hybrid_culture_from_origin_point`, `force_create_head_of_faith_title_Effect`,
`great_project_notify_completion`, `guardian_or_court_tutor_trigger_event`,
`hire_for_court_position_journey_laamp_replacement_effect_2`, `increase_crown_authority_effect`,
`increase_imperial_bureaucracy_effect`, `increase_nomadic_authority_effect`, `increase_tribal_authority_effect`,
`mozarabic_bind_the_faith_to_rome_decision_fundamentalist_path_scripted_effect`,
`mozarabic_bind_the_faith_to_rome_decision_pluralist_path_scripted_effect`,
`mozarabic_bind_the_faith_to_rome_decision_righteous_path_scripted_effect`,
`mozarabic_break_with_rome_decision_fundamentalist_path_scripted_effect`,
`mozarabic_break_with_rome_decision_hof_and_ecumenism_processing_scripted_effect`,
`mozarabic_break_with_rome_decision_pluralist_path_scripted_effect`,
`mozarabic_break_with_rome_decision_righteous_path_scripted_effect`,
`remove_a_criminal_trait_in_faith_effect`, `take_hostage`

### Removed scripted triggers (31)

`can_afford_create_head_of_faith_title_cost_trigger`, `check_tax_collector_aptitude`,
`faith_allows_marriage_consanguinity_trigger`, `faith_is_interesting_heresy_to_state_faith_trigger`,
`guardian_or_court_tutor_2_trait`, `guardian_or_court_tutor_trait`, `has_any_criminal_trait_in_faith_trigger`,
`has_any_shunned_or_criminal_trait_in_faith_trigger`, `has_any_shunned_trait_in_faith_trigger`,
`has_commander_trait_trigger`, `invalid_for_heresy_events`, `is_preferred_heresy`, `is_valid_heresiarch`,
`is_valid_heresy`, `is_wrong_gender_in_faith_trigger`, `murdering_character_is_kinslaying_in_faith_trigger`,
`murdering_character_is_kinslaying_in_my_or_same_dynasty_lieges_faith_trigger`, `no_heretical_hof_faith_trigger`,
`relation_between_characters_is_sodomy_in_my_faith_trigger`, `relation_with_character_is_incestuous_in_faith_trigger`,
`relation_with_character_is_incestuous_in_my_or_lieges_faith_trigger`, `relation_with_character_is_sodomy_in_faith_trigger`,
`relation_with_character_is_sodomy_in_my_or_lieges_faith_trigger`, `relation_with_character_is_sodomy_trigger`,
`relic_war_valid_religious_artefact_trigger`, `relic_war_valid_struggle_artefact_trigger`,
`sexual_activity_with_partner_is_criminal_in_faith_trigger`, `trait_is_criminal_in_faith_trigger`,
`trait_is_shunned_in_faith_trigger`, `trait_is_shunned_or_criminal_in_faith_trigger`,
`trait_is_shunned_or_criminal_in_my_or_lieges_faith_trigger`

### Removed script values (11)

`faith_creation_cost_mult`, `funeral_activity_cost_discount_max_value`, `funeral_activity_cost_discount_medium`,
`funeral_activity_cost_discount_medium_value`, `funeral_activity_cost_discount_min`, `funeral_activity_cost_discount_min_value`,
`legalism_law_cost_modifier`, `odds_skill_contribution_diplomacy_title_value`, `odds_skill_contribution_intrigue_title_value`,
`odds_skill_contribution_learning_target_is_title_value`, `regional_heresy_factor`

---

## 17. Traits, Perks, Men-at-Arms, Artifacts, Titles

| Area | Removed | Added / replacement |
|---|---|---|
| Traits | `scholar` | `erudite` (same stats). Also new: `cleric`, `herald`, `lifestyle_scholar`, two `debug_enable_raiding_*` traits |
| Trait conversion lookup (saves/history) | — | `scholar = erudite`, `eunuch = eunuch_1`, `poet = lifestyle_poet` |
| Lifestyle perks | `scholar_perk` | `erudite_perk` |
| Men-at-arms | `teutonic_knights` | `order_knights`, `order_serjeants` |
| Artifact slots | `trinket_4` | `brooch`, `holy_relic_1`, `holy_relic_2`, `holy_relic_3`; slots support `allow_any_type_or_category`, `holy_site`, `available_holy_site` |
| Landed titles | `d_knights_hospitaler` | `d_knights_hospitaller` (**new `common/landed_titles/title_conversion.lookup`**, format `old = new`) |
| Coats of arms | `d_holy_sepulchre`, `swabian_vaihingen` | +196, including cardinal and clerical-region CoAs; single house/dynasty CoA frame override |
| Dynasties | `castiglione` | +69 |
| Name equivalency | `hunfred_male` | +4 |
| Game concepts | `core_tenet`, `state_faith` | +97 |
| Important actions | `action_reactive_advice_religion` | +33 |
| Tutorial lessons | `reactive_advice_religion` | +6 |
| Messages | `holy_order_destroyed` | +46 |

---

## 18. Portraits & Genes

- **12 new accessory gene templates:** `ep1_indian_book_big_left`, `ep1_mediterranean_book_big_left`,
  `ep1_mena_book_big_left`, `ep1_western_book_big_left`, `pam_bible_ornate_01_a_left`,
  `pam_bible_ornate_open_01_left`, `pam_crozier_catholic_basic_01`, `pam_crozier_catholic_ornate_01`,
  `pam_crozier_orthodox_basic_01`, `pam_crozier_orthodox_ornate_01`, `pam_situla_01_a`, `tgp_chinese_book_closed_left`.
- **`eye_accessory` reordered:** a new `no_eyes` entry was inserted first, and `normal_eyes` now declares `index = 1`.
- Changed gene files: `01_genes_morph`, `02_genes_accessories_misc`, `03_…hairstyles`, `04_…beards`,
  `05_…clothes` (524 lines differ), `06_…headgear` (313), `07_…misc`, `08_…visual_traits`.
- `common/ethnicities/00_ethnicities_templates.txt` no longer sets `eye_accessory` / `eyelashes_accessory`.
- About 25 files in `gfx/portraits/portrait_modifiers/` changed (clothes, headgear, religious clothing, animation props).
- **Accessory gene index limit raised from 255 to 2,147,483,647.** Portrait DNA keeps accessory gene groups above 255.
- New `portrait_modifier_pack` and `portrait_modifier_set` for portrait animations, a no-portrait blend
  shape for all portrait types, and a fix for a crash in the DNA accessory tooltip on script-declared portrait modifiers.
- Dead characters: `DEAD_PORTRAIT_AGE_THRESHOLD`, `DEAD_PORTRAIT_APPARENT_AGE` defines.
- 667 new files under `gfx/models`, 21 under `gfx/portraits`.

---

## 19. GUI & Data Binding

- **Removed file:** `gui/window_faith_creation.gui`, replaced by **`gui/window_rite_creation.gui`**.
- **New windows:** `window_cardinals.gui`, `window_geographical_regions.gui`, `window_holy_site.gui`,
  `window_holy_site_creation.gui`, `window_organization.gui`, `window_pam_christian_church.gui`,
  `window_personal_beliefs.gui`, `window_puppet_selection.gui`, `window_rite_creation.gui`,
  `window_title_selector.gui`, `shared/succession_election_widgets.gui`, `shared/vertical_progressbars.gui`.
- **New decision widgets:** select artifact, character, holy site, realm county, rite, tenet, title, a
  generic title selector, and petition head of faith. The new `select_scope_object` decision controller
  lets decisions offer lists of any object type. Open one pre-selected with `PdxGuiWidget.MakeDecisionTypeWithParam`.
- **New event window widgets:** 22, including rite, rite founder, new pope, name pope, the Christian Church
  situation info, and new VFX backgrounds (stained glass, candlelight, night scene, double vision).
- **120 vanilla `.gui` files changed.** Top-level list: `frontend_bookmarks`, `frontend_ingame_menu`, `frontend_load`, `frontend_main`,
  `hud`, `hud_bottom`, `hud_notification_templates`, `interaction_*` (concubine, confirmation, council_task,
  court_task, create_claimant_faction, declare_war, grant_titles, marriage, menu_window, revoke_title,
  templates), `map_icon_layer`, `multiplayer_lobby`, `texticons`, `texticons_religion`, and `window_*` for accolade, activity,
  activity_guest_list, activity_list, activity_planner, admin_vassal_detail, appoint_tax_collector,
  army_select_commander, artifact_details, barbershop, battle_summary, character, character_filter,
  character_lifestyle, confederation, council, county_view, court_positions, culture, decisions,
  decisions_detail, diverge_culture, domicile, dynasty_legacy, dynasty_tree, factions, faith,
  faith_conversion, find_title, find_vassal, government_administration, great_project, hired_troops_detail,
  hybridize_culture, intrigue, inventory, knights, ledger, manage_tax_slots, message_popup,
  message_settings, military, my_realm, plan_great_project, replace_pillar, ruler_designer,
  ruler_designer_load, silk_road, situation, situation_list, situation_participation,
  situation_participation_debug, struggle_involvement, succession_event, tax_slot_vassals,
  tgp_dynastic_cycle, the_great_steppe, title, title_appointment, title_claimants, title_election,
  title_history, treasury_budget_change, war_overview. The rest are in subfolders.
- **Removed templates and types:** `Button_Close_Select_Faith`, `Button_Select_Faith`,
  `Placeholder_Button_Soundeffect`, `faith_tooltip_core_tenents_extra_info`,
  `faith_tooltip_ruler_designer_extra_info`, `religion_tooltip`, `button_religion_icon_ruler_designer`,
  `container_tenet_item`, `faith_tooltip_core_tenents`, `fervor_container_vbox`, `vbox_secret_item`,
  `widget_doctrine_item`, `widget_succession_candidate_banner_small`, `widget_succession_candidate_item`
  (plus 26 named widgets).
- **Data functions:** `HasCost` is gone from vanilla GUI and `HasCostOrBreakdown` is the replacement pattern.
  New: `HasMechanic`, `IsEminentHolySite`, `GetDivergenceBreakdown`, `CalcDivergenceScore`, `SetSelectedRite`,
  `GetDoctrineType`, `GetTenetBackgroundSmallIcon`, `SelectArtifact`, `GetTopParticipantGroupByKey`,
  `OpenSelectionForDoctrineIndex`, `MakeDecisionTypeWithParam`, `GetHeaderBackground`.
- **Data-binding functions:** `And2`–`And8`, `Or2`–`Or8`, `Nor2`–`Nor8`, `Nand2`–`Nand8`, `Nor()`, `Nand()`,
  the `(int_bool)0` cast, and `DataModelJoinList( list, 'loc_key_item', 'loc_key_separator' )` (exposes `MODEL_SIZE`,
  `ITEM_INDEX`, `ITEM_NUM`, `IS_LAST_INDEX` in localization). `Custom()` customizable localization now works on
  `government_type`, `tenet` and `rite_type`.
- **New data_binding files:** `efg_data_bindings.txt`, `pam_data_bindings.txt`. Changed: `character_grammar_macros.txt`,
  `ep2_tournament_data_bindings.txt`, `ep3_data_bindings.txt`, `gui_macros.txt`.
- Event windows: characters can be depth-stacked per event with `stacking_order = X`. Red blocker breakdowns show nested elements at any depth.

---

## 20. Defines

**Removed (4):** `CONVERGENCE_DELAY`, `FAITH_CREATION_FERVOR_DISCOUNT_MAX`,
`FAITH_CREATION_FERVOR_DISCOUNT_PER_MISSING_FERVOR`, `MINIMUM_FAITH_SIZE_FERVOR_MODIFIER`.

**Added (44):**
`AI_APPOINTMENT_MAX_SKIP_PERMILLE`, `AI_APPOINTMENT_REALM_SIZE_SCALEDOWN_FULL`,
`AI_APPOINTMENT_REALM_SIZE_SCALEDOWN_START`, `ARTIFACT_DURABILITY_DECAY_ENSHRINED`, `BASE_PERSONAL_TENETS_CAP`,
`CREATE_HOLY_SITE_RARITY_REQUIRED`, `DEAD_PORTRAIT_AGE_THRESHOLD`, `DEAD_PORTRAIT_APPARENT_AGE`,
`FAITH_CORE_TENETS_CAP`, `FAITH_EMINENT_HOLY_SITES_MAX_DEFAULT`, `FAITH_EMINENT_HOLY_SITES_MIN_DEFAULT`,
`FAITH_HOLY_SITES_MAX_DEFAULT`, `FAITH_HOLY_SITES_MIN_DEFAULT`, `FREE_HERETICAL_RITES_BEFORE_DEPLETION`,
`HERESY_FERVOR_DEPLETION_MULTIPLIER`, `INTENDED_MAX_PERSONALITY_TRAITS`, `LOW_FERVOR_THREASHOLD`,
`MAX_FAITH_SIZE_PER_RITE`, `MAX_FERVOR_LOSS_PER_DIVERGENT_RITE`, `MAX_HOLY_SITE_RARITY_TOTAL`,
`MAX_RELICS_FROM_SAINT`, `MAX_SPIRITUAL_FULFILLMENT`, `MAX_VIRTUES_SINS`, `MIN_FULFILLMENT_CHANGE_FOR_EFFECT`,
`MIN_SPIRITUAL_FULFILLMENT`, `MONTHLY_RITE_CONVERSION_PROCESS_PER_NEIGHBORING_COUNTY`,
`OVER_NEEDED_RITES_FERVOR_LOSS_PROTECTION_MULT`, `PERSONAL_TENET_CHANGE_COOLDOWN_DAYS`, `PIOUS_CLERGY`,
`REQUIRE_SAME_CHAPLAIN_RITE_FOR_REGION_CONVERSION`, `RITE_COLOR_DEVIATION`,
`RITE_CREATION_DIVERGENCE_FAITH_THRESHOLD`, `RITE_CREATION_FERVOR_DISCOUNT_MAX`,
`RITE_CREATION_FERVOR_DISCOUNT_PER_MISSING_FERVOR`, `RITE_CREATION_PIETY_COST`,
`RITE_DIVERGENCE_GRACE_THRESHOLD`, `RITE_DIVERGENCE_HERETICAL_THRESHOLD`, `RITE_DIVERGENCE_HOSTILITY_THRESHOLD`,
`SPIRITUAL_FULFILLMENT_FOR_STRESS_DIVIDE_FACTOR`, `SPIRITUAL_FULFILLMENT_MULT_CORE_TENET`,
`SPIRITUAL_FULFILLMENT_MULT_PERMITTED_TENET`, `SPIRITUAL_FULFILLMENT_MULT_PERSONAL_TENET`,
`SPIRITUAL_FULFILLMENT_MULT_RELIGION_TRAIT`, `TENET_STATUS_DIVERGENCE`.

**Best practice:** put only the defines you change in a uniquely named file, e.g.
`common/defines/zz_mymod_defines.txt` with `NGame = { ... }` blocks. Never ship a copy of `00_defines.txt`.

---

## 21. Localization

- English: 282,564 → 299,447 keys (**+17,499 / −616**).
- About 695 new files and about 4,092 changed files across all languages; 88 non-English files removed (mostly stale dev and test files).
- **Morpheme strip rules** (`common/morpheme_strip_rules/`) power a new formatter, `$VALUE|:rule_name$`
  (e.g. `[Character.GetName|:english_rite_ism]ism`). It combines with other formatters (`|U:rule`).
- Notification messages can display Rite, Faith, Doctrine and Tenet icons.
- New `localization/workbench.json`.

---

## 22. Map

- `map_data` binaries (provinces, heightmap, rivers, `definition.csv`, `default.map`, adjacencies) are **identical** in 1.19.0.6 and 1.20.0.2.
- `map_data/geographical_regions/geographical_region.txt` changed: 70+ new regions (`et_867_*`,
  `et_1066_*`, `custom_slavic_rite_mission_region`, `dlc_mpo_steppe_north_east_asia_expansion`, …).
- Geographical regions can include **`hegemonies = { }`** (new `_geographical_regions.info`).
- River graphics now hot-reload when `rivers.png` changes (gameplay data such as river crossings does not).

---

## 23. Other New Modding Features

- `id_override_priority` on event definitions lets you override one vanilla event without replacing the file.
- `history_override_priority` on character history (see [§8](#8-history-files)).
- **Fallback inheritance:** a title can name a character or title that inherits it when no normal heir exists.
- Activities can configure tenets and doctrines instead of activity options, and have a triggered `header_background`; invite rules can be locked.
- Situations: `gui_tags` on participant groups, extra texture on phases, and situation decisions registered via the `.info`.
- Great Projects: `founder_heir` support.
- Appointment types: `use_investment_cap`. Election succession supports MTTH and script-value syntax.
- Flavorization: restrict by title kind (heads of faith, clerical regions, noble families), require a Rite,
  apply filters to the **lessee**, and a saint category.
- Bookmark characters can be lowborn.
- A parameter can conditionally allow clergy to marry despite their Rite's doctrines.
- New command line option `-bookmark` (see the new `game/_commandline_options.info`).
- Object explorer and inspector query history (Ctrl+Up / Ctrl+Down).
- Clearer trigger/effect perspective error messages.
- A local copy of a mod now loads even when the Workshop version is installed.
- Much less error.log bloat from saves made with incompatible playsets.
- Hot-reload fixes: on-action DB crash, cumulative-law duplication, war history duplication; holy site types are now hot-reloadable.
- An assert fires when a dynamic modifier shares a token with a static one.
- Coat of Arms modding override settings; government flag realm-mask offsets and scales can be overridden centrally.

---

## 24. Complete Removed-Key Reference

<details>
<summary><b>Doctrine types removed (73 — all moved to <code>tenet_types</code>)</b></summary>

`tenet_adaptive` `tenet_adorcism` `tenet_alexandrian_catechism` `tenet_ancestor_worship` `tenet_aniconism`
`tenet_armed_pilgrimages` `tenet_asceticism` `tenet_astrology` `tenet_benevolent_governance` `tenet_bhakti`
`tenet_carnal_exaltation` `tenet_christian_syncretism` `tenet_communal_identity` `tenet_communal_possessions`
`tenet_communion` `tenet_consolamentum` `tenet_cranial_trophies` `tenet_cthonic_redoubts` `tenet_dharmic_pacifism`
`tenet_divine_marriage` `tenet_eastern_syncretism` `tenet_esotericism` `tenet_exaltation_of_pain`
`tenet_extinction_of_dharma` `tenet_false_conversion_sanction` `tenet_filial_piety` `tenet_fp3_fedayeen`
`tenet_gnosticism` `tenet_gruesome_festivals` `tenet_harmonious_society` `tenet_hedonistic` `tenet_household_gods`
`tenet_human_sacrifice` `tenet_inner_journey` `tenet_islamic_syncretism` `tenet_jewish_syncretism` `tenet_legalism`
`tenet_literalism` `tenet_megaliths` `tenet_mendicant_preachers` `tenet_monasticism` `tenet_mountain_worship`
`tenet_mystical_birthright` `tenet_natural_primitivism` `tenet_no_mind` `tenet_pacifism` `tenet_pastoral_isolation`
`tenet_pentarchy` `tenet_polyamory` `tenet_preservation` `tenet_pure_land` `tenet_pursuit_of_knowledge`
`tenet_pursuit_of_power` `tenet_reincarnation` `tenet_religious_legal_pronouncements` `tenet_rite`
`tenet_ritual_cannibalism` `tenet_ritual_celebrations` `tenet_ritual_hospitality` `tenet_sacred_childbirth`
`tenet_sacred_destruction` `tenet_sacred_shadows` `tenet_sacrificial_ceremonies` `tenet_sanctity_of_nature`
`tenet_sinitic_syncretism` `tenet_struggle_submission` `tenet_sun_worship` `tenet_takamin`
`tenet_tax_nonbelievers` `tenet_unreformed_syncretism` `tenet_unrelenting_faith` `tenet_vows_of_poverty`
`tenet_warmonger`

Doctrine group removed: `doctrine_core_tenets`.
</details>

<details>
<summary><b>Trigger localization removed (10)</b></summary>

`HOF_is_ai_tt`, `can_execute_decision`, `hof_interaction_unreformed_faith`,
`incompatible_tenet_monasticism_trigger`, `landed_title_culture_group_equal`,
`mozarabic_fate_county_count.need_at_least_twenty_same_faith_counties`, `number_of_personality_traits`,
`number_of_personality_traits_in_common`, `rites_of_passage_trigger_chaplain`, `top_liege_equal`.
The file `common/trigger_localization/00_scope_comparison_triggers_l_english.txt` was renamed to `00_scope_comparison_triggers.txt`.
</details>

<details>
<summary><b>Effect localization removed (5)</b></summary>

`activate_holy_site`, `deactivate_holy_site`, `stewardship_haggle_cheap_trinket_tt`,
`stewardship_haggle_expensive_trinket_tt`, `stewardship_haggle_helpful_trinket_tt`.
</details>

<details>
<summary><b>Modifier definition formats removed (7)</b></summary>

`ashari_opinion`, `faith_creation_piety_cost_add`, `faith_creation_piety_cost_mult`, `imami_opinion`,
`maturidi_opinion`, `mutazila_opinion`, `muwalladi_opinion`.
</details>

<details>
<summary><b>Other removed database keys</b></summary>

| Database | Removed keys |
|---|---|
| `common/decisions` | `ai_create_head_of_faith_decision`, `appoint_a_righteous_caliph_decision`, `mozarabic_bind_the_faith_to_rome_decision`, `mozarabic_break_with_rome_decision` |
| `common/modifiers` | `prepare_travels_modifier`, `rejected_from_marriage_bed_modifier` |
| `common/scripted_guis` | `create_head_of_faith`, `recreate_head_of_faith` |
| `common/scripted_rules` | `faith_creation` |
| `common/traits` | `scholar` |
| `common/lifestyle_perks` | `scholar_perk` |
| `common/men_at_arms_types` | `teutonic_knights` |
| `common/artifacts/slots` | `trinket_4` |
| `common/landed_titles` | `d_knights_hospitaler` (renamed) |
| `common/ethnicities` | `eye_accessory`, `eyelashes_accessory` template entries |
| `common/coat_of_arms` | `d_holy_sepulchre`, `swabian_vaihingen` |
| `common/dynasties` | `castiglione` |
| `common/culture/name_equivalency` | `hunfred_male` |
| `common/game_concepts` | `core_tenet`, `state_faith` |
| `common/important_actions` | `action_reactive_advice_religion` |
| `common/tutorial_lessons` | `reactive_advice_religion` |
| `common/messages` | `holy_order_destroyed` |
| `common/on_action` | `on_character_faith_change`, `faith_fervor_events_pulse` |
| `common/religion/doctrine_types` | file `30_core_tenets.txt` |
| `events/` | file `religion_events/fervor_events.txt`; events `fervor.1001`, `fervor.1002`, `fervor.2001`, `fervor.2002`, `heresy.0001`, `heresy.0005`, `heresy.0010`, `heresy.0011`, 44 × `learn_commander_trait.*`, `iberia_north_africa.0101/0102/0111/0112`, `east_europe.0015`, `death_management.0098` |
| `gui/` | file `window_faith_creation.gui` |

</details>

---

## 25. Validating Your Mod

1. Launch with `-debug_mode` (and optionally `-bookmark=<key>` to jump straight to a start date).
2. Start a **new** game. Don't load a 1.19 save.
3. Open `Documents/Paradox Interactive/Crusader Kings III/logs/error.log` and search for these signatures:

| Log signature | Meaning | Section |
|---|---|---|
| `doesn't specify a doctrine_group_type` | Doctrine is missing its group | [§7](#7-doctrines-and-doctrine-groups) |
| `Not found in database class CDoctrineTypeDatabase` | `has_/add_/remove_doctrine` with a tenet (or deleted doctrine) | [§6](#6-tenets-are-now-their-own-database) |
| `Failed to find 'scholar' from database for link 'trait'` | Renamed trait | [§17](#17-traits-perks-men-at-arms-artifacts-titles) |
| `Invalid database object 'scholar_perk'` | Renamed perk | [§17](#17-traits-perks-men-at-arms-artifacts-titles) |
| `Not found in database class CFaithTypeDatabase` / invalid faith | Faith key became a Rite | [§5](#5-faith-keys-that-became-rites-lookup-table) |
| `Not found in database class CGovernmentTypeDatabase` | Government key missing or renamed | [§10](#10-governments) |
| `Not found in database class CMenAtArmsTypeDatabase` | e.g. `teutonic_knights` | [§17](#17-traits-perks-men-at-arms-artifacts-titles) |
| `Unknown effect type` / `Unknown trigger type` | Removed scripted effect or trigger | [§16](#16-script-api-renamed-removed-added) |
| `Failed to fetch a valid landed title` | Removed or renamed title | [§17](#17-traits-perks-men-at-arms-artifacts-titles) |

Quick grep for the most common 1.19-isms in a mod folder:

```bash
grep -rnE 'has_doctrine = tenet_|add_doctrine = tenet_|remove_doctrine = tenet_' .
grep -rnE 'on_character_faith_change|set_state_faith|create_holy_order_effect|lease_out_to\b' .
grep -rnE '(increase|decrease)_(crown|tribal|nomadic)_authority_effect|(increase|decrease)_imperial_bureaucracy_effect' .
grep -rnE '_in_(my_or_lieges_)?faith_trigger|has_commander_trait_trigger|doctrine_selection_pair|doctrine_background_icon' .
grep -rnE '\bscholar(_perk)?\b|teutonic_knights|trinket_4|d_knights_hospitaler\b' .
grep -rnE 'faith:(ashari|maturidi|mutazila|muwalladi|coptic|imami|ismaili|theravada|mahayana|vajrayana|shinto|insular_celtic|mozarabic_church|rabbinism)\b' .
grep -rnE 'religious = yes|administrative = yes' common/governments
grep -rn  'faiths = {' common/religion
grep -rLE 'law_group_type' common/laws/*.txt      # law files still in 1.19 nested format
```

---

## 26. Methodology & Sources

- **Baseline:** `base/game` in this repository at **1.19.0.6 "Scribe"** (`base/.ck3-version.json`).
- **Target:** a clean Steam install of **1.20.0.2 "Crozier"** (`launcher-settings.json` → `rawVersion: 1.20.0.2`).
- **File diff:** full tree comparison. Text files were compared after normalizing CRLF/LF, UTF-8 BOM and trailing whitespace.
  Binary map files were compared by SHA-256 against this repo's placeholder metadata. The `dlc/` folder and
  launcher/root binaries were excluded, because they depend on the install.
- **Key diff:** every top-level database key in `common/`, `events/`, `map_data/` and `history/` was compared
  between versions to produce the removed and added lists above.
- **Usage diff:** vanilla script usage of each trigger, effect, scope and data function was counted in both versions to find renames.
- **Field test:** a large 1.19-era mod was loaded on 1.20.0.2, and its `error.log` (17,866 errors) was
  categorized. Items marked ✔ were confirmed there.
- **Official patch notes:** [Now Available: By God Alone & 1.20.0 "Crozier" Update](https://store.steampowered.com/news/app/1158310/view/1845383656377574)
  (Steam announcement, *Modding* section).
- **Vanilla `.info` documentation** shipped with 1.20.0.2: `_faith_types.info`, `_rite_types.info`,
  `_tenet_types.info`, `_doctrine_types.info`, `_doctrine_group_types.info`, `_doctrine_category_types.info`,
  `_religion_types.info`, `_laws.info`, `_law_groups.info`, `_governments.info`, `_holy_orders.info`,
  `_puppet_types.info`, `_puppet_actions.info`, `_spiritual_fulfillment_type.info`, `_slots.info`,
  `_geographical_regions.info`, `00_morpheme_strip_rules.info`, `history/_characters.info`,
  `history/_provinces.info`, `history/faiths/_faith_history.info`, `_on_actions.info`, `_commandline_options.info`.

> Risk ratings are an engineering assessment meant to help you prioritize. They are not a promise that a
> given change will or will not crash your game. When in doubt, check `error.log` on a fresh start.
> Corrections and additions are welcome via pull request.
