# Hubs: places with several charging sites

This is the group's one copy of what a hub is. Each repo documents only its own part and links here.

**Status: proposed, not built.** Each repo's part of the work is in its plan: [nap-dashboard `docs/plan.md`](https://github.com/CarregaErvilhas/nap-dashboard/blob/main/docs/plan.md#hubs) and [OlhaOPosto `Docs/plan.md`](https://github.com/CarregaErvilhas/OlhaOPosto/blob/main/Docs/plan.md). Change this file in the same change as a rule below.

## Why

No source publishes a place with several sites. Each operator publishes its own sites in the NAP, and many split one place into one site per charger.

For example, Centro Comercial Colombo has 16 NAP sites from 3 operators, with 60 sockets. Searching for it returns one result per charger, and it misses the sites whose name is only their code.

A hub groups the sites of one place, across operators.

## Who does what

| Repo | Role |
| --- | --- |
| nap-dashboard | The only place that decides what a hub is: it builds hubs from OpenStreetMap into the network model (`hubs`, and `hub` on each site). It also shows them on its site. |
| OlhaOPosto | Imports hubs from the model. It holds the hub photos (the "Local" category), the static link `pictures.ocarroeletrico.com/HUB-….jpg`, the hub page, the API and the MCP. It keeps a dissolved hub. |
| ohmigo-mcp | Reads `hubs` from the model: it finds a hub by name and compares operators at one hub. |
| AmpereArchive | Nothing. The archive stays per `SITE/POINT`; a reader adds up the sites of a hub. |

The model format, the pipeline and the screens are in each repo, not here.

## Definition

- **A hub is an OpenStreetMap area that holds two or more NAP sites.**
- **The area is a place of one of these kinds**, in this order of importance:

  | # | OSM tag | Place |
  | --- | --- | --- |
  | 1 | `shop=mall` | shopping centre |
  | 2 | `amenity=hospital` | hospital |
  | 3 | `amenity=university` | university |
  | 4 | `aeroway=aerodrome` | airport |
  | 5 | `leisure=stadium` | stadium |
  | 6 | `highway=services` | motorway services (new) |
  | 7 | `landuse=retail` | retail area |
  | 8 | `shop=supermarket` | supermarket |
  | 9 | `amenity=fuel`, as an area | fuel station (new) |
  | 10 | `amenity=parking` | car park |
  | 11 | `tourism=hotel` | hotel |

  The list is nap-dashboard's `osm_places.py` `KINDS`, plus the two kinds marked new.
- **A site belongs to the place whose outline holds its position.**
  - The position is the network model's, which is OpenStreetMap's.
  - Each kind has a small margin, the one nap-dashboard already uses to say which place a site is at.
- **A site inside two places goes to the more important one.** A car park inside a shopping centre goes to the shopping centre.
  - Exception: if that place's sites lie more than about 300 m apart (a campus, an airport), the site goes to the next, smaller place.
- **A site with no place is in no hub.** Nothing is grouped by distance alone: such a group would have no name and no stable id. Sites close to each other are shown as nearby, never as a hub.
- **Sites under construction** (OpenStreetMap only, ids `OSM-…`) inside the place are listed with the hub, but they don't count toward the two.

## Id

`HUB-` followed by the OpenStreetMap element: `w` for a way, `r` for a relation, then its number. For example, `HUB-w42645796` is Centro Comercial Colombo, `way/42645796`.

- Renaming the place keeps the id.
- An id never changes meaning. It always names that OpenStreetMap element, so a hub's static link and page always point to the same place.

## Name

- **The place's OpenStreetMap `name`**, and `name:en` in English when present.
- **A place with no name** shows its kind and the NAP address of a site, for example "Parque de estacionamento · Avenida Lusíada, Lisboa", with a link to add the name on OpenStreetMap.
- **Only nap-dashboard decides the name.** OlhaOPosto and ohmigo read it and never edit it.
- **Two hubs within 1 km with the same name** (both sides of one motorway services area): nap-dashboard warns, and someone resolves it on OpenStreetMap or in the manual file.

## Zones

When the sites of a hub have different OpenStreetMap `level` values, the hub shows them as zones, for example "Piso -3". A large place that is right is shown in zones and never split.

## Manual exceptions

A small file in nap-dashboard, each entry with `source` and `checkedAt`, holds only two kinds of exception:

- a name override;
- a site to exclude from a hub.

Everything else is fixed on OpenStreetMap.

## Splitting a hub

**Fix OpenStreetMap, then wait for the next refresh of the model.** Map only what exists on the ground: never draw or change an area just to group our sites.

| Case | What to do |
| --- | --- |
| The area holds places that are really separate | Split the area, or map the smaller places: each car park, each side of a motorway |
| The place is right, only large | Nothing. The hub shows its zones |
| A site that should not be in the hub | Fix the site's position on OpenStreetMap, or exclude it in the manual file |

Real case: at Grândola, OpenStreetMap maps only the south-to-north side of the A2 as `highway=services` (`way/208645443`). That side's sites form a hub. The other side's sites don't, until someone maps that side's area. That is the right result, because nobody can cross the motorway.

## When a hub changes

This is the contract for every reader of the model.

- **The area that remains keeps its id**, and new areas get new ids.
- **Sites never change** because of a hub. They keep their ids, pages, photos and static links; only their `hub` field changes.
- **A hub that leaves the model** (area deleted, or fewer than two sites) is never deleted by a reader. OlhaOPosto marks it dissolved:
  - its static link keeps serving its last main photo, because a link never changes;
  - its page says which hubs replaced it;
  - its photos go to moderation, to be moved to the new hubs.
- **The same place redrawn on OpenStreetMap,** with a new id and the same sites: a moderator confirms an alias, and the old id answers `301`.

## OpenStreetMap: what we read and write

**What we read:**

- **The charging stations** (`amenity=charging_station`, `man_made=charge_point`) come from the Overpass dump of the community map "Postos de Carregamento v2.1". They are joined to the NAP by `ref` or `ref:EU:EVSE`, as described in nap-dashboard's [`docs/network-model.md`](https://github.com/CarregaErvilhas/nap-dashboard/blob/main/docs/network-model.md#joining-osm-to-the-nap).
- **The place outlines** come from Geofabrik's Portugal extract, once a day.

**What we write:**

- **Nothing automatic today.** OlhaOPosto shows the `panoramax=` tag of a station's main photo and a button to open the element in iD, for a person to add by hand.
- **A photo bot may come later,** only after the community approves it. It would only add `panoramax=` to elements already linked by `ref`.
- **Never geometry.**

**Credit:** "© OpenStreetMap contributors", under ODbL, wherever OpenStreetMap data is shown or published.

## Open questions for the OpenStreetMap community

- Should each side of a motorway services area be its own `highway=services` area, and does its `name` carry the direction?
- Is there a convention for several operators' stations at one place (a `type=site` relation, for example) that we should read, instead of inferring the hub from the outline?
- For zones, is `level` on `amenity=charging_station` enough, or should each car park floor be its own `amenity=parking`?
- Should hub photos (overview, entrance, signs) be linked to any OpenStreetMap element?
