# Hubs and station areas: grouping charging sites

This is the group's one copy of how charging sites are grouped into station areas and hubs. Each repo documents only its own part and links here.

**Status: proposed, not built.** Each repo's part of the work is in its plan: [nap-dashboard `docs/plan.md`](https://github.com/CarregaErvilhas/nap-dashboard/blob/main/docs/plan.md#hubs-and-station-areas) and [OlhaOPosto `Docs/plan.md`](https://github.com/CarregaErvilhas/OlhaOPosto/blob/main/Docs/plan.md). Change this file in the same change as a rule below.

## Why

No operator publishes a place, only its own sites, and the NAP shows two problems:

- **One operator splits one place into many sites.** For example, the 15 Helexia sites at Auchan Cascais have one socket each.
- **Several operators share one place.** Centro Comercial Colombo has 16 NAP sites from 3 operators.

A search returns one result per site in both cases.

## Four levels

| Level (pt) | What it is | Where it comes from | Id |
| --- | --- | --- | --- |
| **Hub** | A place (shopping centre, motorway services, car park…) that holds several station areas or sites | An OpenStreetMap place area | `HUB-` + element, `HUB-w42645796` |
| **Station area** (estação) | The `amenity=charging_station` area that OpenStreetMap mappers draw around a group of chargers, almost always one operator's | OpenStreetMap | `EST-` + element, `EST-w1355046979` |
| **Site** (posto) | One NAP site: one charger, mapped in OpenStreetMap as a `man_made=charge_point` whose `ref` is the NAP code | NAP | NAP code |
| **Socket** (tomada) | One EVSE | NAP | EVSE code |

A **location** (local) is the top level that groups a site: its hub, or its station area when there is no hub. A site in neither stands alone.

**What this means in practice,** from one day's OpenStreetMap data:

- There are about 2 000 station areas. About 1 400 of them hold two or more NAP sites, which covers almost 3 900 sites.
- Only about 50 of those station areas hold more than one operator.

So station areas fix most of "one place, many results". Hubs are for several operators at one place.

## Who does what

| Repo | Role |
| --- | --- |
| nap-dashboard | The only place that decides station areas and hubs. It builds them from OpenStreetMap into the network model (`stationAreas`, `hubs`, and `stationArea`/`hub` on each site) and shows them on its site. It lists the OpenStreetMap disagreements it finds. |
| OlhaOPosto | Imports both levels from the model. It holds the location photos (the "Local" category), the static links `pictures.ocarroeletrico.com/HUB-….jpg` and `EST-….jpg`, and the location page, API and MCP. It keeps a dissolved location. |
| ohmigo-mcp | Reads both levels from the model: it finds a location by name and compares operators at one hub. |
| AmpereArchive | Nothing. The archive stays per `SITE/POINT`; a reader adds up the sites of a location. |

The model format, the pipeline and the screens are in each repo, not here.

## Station areas

**Source:** the `amenity=charging_station` ways in `Todos.json`, the Overpass dump behind the community map "Postos de Carregamento v2.1". The dump includes the ways' nodes, so each area's outline is known.

**Members.** These are the two methods the community suggested, used together:

1. **By `ref`.** The area lists its sites' NAP codes: `ref=ABC-00001;ABC-00002`, with a `name` like `ABC-00001/2`. Each site's `man_made=charge_point` node carries one of those codes.
2. **By geometry.** Every `man_made=charge_point` node inside the area's outline is a member, including `disused:` and `construction:` ones. This catches what has no NAP code:
   - Tesla bays: the area's `ref` is the NAP's long site id, and the bays carry numbers like `5C`;
   - chargers not yet published in the NAP, such as some Continente ones.

The outline test is a point-in-polygon in the standard library, as `osm_places.py` already does: no new dependency.

**Disagreements.** A code in the area's `ref` list may have no node inside the area, or a node inside may carry a code missing from the list. The pipeline lists these for the OpenStreetMap community (typos, overlapping areas) and never corrects them itself. Tesla bay numbers are not disagreements.

**Rules:**

- **It groups only with two or more NAP sites.** An area around a single site is just that site's element.
- **Name.** The area's own `name` is the mappers' code shorthand (`GDL-00008/27/28/…`), so it isn't shown. In order, the shown name is:
  1. the name of the place it is in, from the place kinds below, even when that place is not a hub;
  2. the NAP name its sites share;
  3. the operator and the NAP address.
- **Id:** `EST-w1355046979` is Auchan Cascais (`way/1355046979`), 15 Helexia sites.

## Hubs

**A hub is an OpenStreetMap place that holds at least two units.** A unit is a station area, or a site that is in no station area.

**The place is an area of one of these kinds,** in this order of importance:

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

**Rules:**

- **What gets tested against the place.** A station area belongs to the place that holds its outline's centre. A loose site belongs by its position, which is the model's, so OpenStreetMap's. Testing the area is more robust than testing each site, whose NAP or OpenStreetMap point can sit outside its own area.
- **A unit inside two places goes to the more important one.** If that place's units lie more than about 300 m apart (a campus, an airport), it goes to the next, smaller place instead.
- **A place with one unit is not a hub.** Its station area takes the place's name.
- **A unit in no place is in no hub.** Nothing is grouped by distance alone.
- **Sites under construction** (OpenStreetMap only, ids `OSM-…`) are listed where they fall, but don't count toward the two units.
- **Id:** `HUB-w42645796` is Centro Comercial Colombo (`way/42645796`). Renaming the place keeps the id.
- **Name:** the place's OpenStreetMap `name`, and `name:en` in English when present. A place with no name shows its kind and a site's NAP address, with a link to add the name on OpenStreetMap.
- **Zones:** a hub shows its units as zones, each with its `level` when mapped. Nothing changes the hub itself.
- **Two hubs within 1 km with the same name:** nap-dashboard warns, and someone resolves it on OpenStreetMap or in the manual file.

## Examples, checked against OpenStreetMap

**Centro Comercial Colombo is a hub** (`way/42645796`), with three zones:

- EDP: one station area (`way/1360189671`) with 10 sites;
- Galp: one station area (`way/1494355946`) with 2 sites;
- CapWatt: 4 sites, each mapped as its own area with `level=-3`, so 4 loose units.

**Auchan Cascais is a station area,** not a hub: one area with 15 Helexia sites.

**Grândola, on the A2, is two station areas** of 8 EDP sites each, one per side of the motorway.

- OpenStreetMap maps only the south-to-north side as `highway=services` (`way/208645443`). That side's station area takes the name "Area de Serviço de Grândola".
- The other side's station area has no place, so it shows its NAP name.
- Neither side is a hub, because each side holds one unit.

## Manual exceptions

A small file in nap-dashboard, each entry with `source` and `checkedAt`, holds only two kinds of exception:

- a name override;
- a site to exclude from a station area or hub.

Everything else is fixed on OpenStreetMap.

## Fixing a grouping

**Fix OpenStreetMap, then wait for the next refresh of the model.** Map only what exists on the ground: never draw or change an area just to group our sites.

| Case | What to do |
| --- | --- |
| A station area misses a site, or lists one that is elsewhere | Fix the area's `ref` or its outline |
| A place holds things that are really separate | Split it, or map the smaller places: each car park, each side of a motorway |
| A place is right, only large | Nothing. The hub shows its zones |
| A site that should not be in a location | Fix the site's position on OpenStreetMap, or exclude it in the manual file |

## When a location changes

This is the contract for every reader of the model, for both levels.

- **The area that remains keeps its id**, and new areas get new ids.
- **Sites never change** because of a location. They keep their ids, pages, photos and static links; only their `stationArea` and `hub` fields change.
- **A location that leaves the model** (area deleted, fewer than two sites or units, or a station area now inside a new hub) is never deleted by a reader. OlhaOPosto marks it dissolved:
  - its static link keeps serving its last main photo, because a link never changes;
  - its page says which locations replaced it;
  - its photos go to moderation, to be moved.
- **The same place or area redrawn on OpenStreetMap,** with a new id and the same sites: a moderator confirms an alias, and the old id answers `301`.

## OpenStreetMap: what we read and write

**What we read:**

- **Charging stations:** the `amenity=charging_station` areas and nodes, and the `man_made=charge_point` nodes, all from `Todos.json`. Sites are joined to the NAP by `ref` or `ref:EU:EVSE`, as described in nap-dashboard's [`docs/network-model.md`](https://github.com/CarregaErvilhas/nap-dashboard/blob/main/docs/network-model.md#joining-osm-to-the-nap).
- **Place outlines:** from Geofabrik's Portugal extract, once a day.

**What we write:**

- **Nothing automatic today.** OlhaOPosto shows the `panoramax=` tag of a site's main photo and a button to open the element in iD, for a person to add by hand.
- **A photo bot may come later,** only after the community approves it. It would only add `panoramax=` to elements already linked by `ref`.
- **Never geometry.**

**Credit:** "© OpenStreetMap contributors", under ODbL, wherever OpenStreetMap data is shown or published.

## Open questions for the OpenStreetMap community

- Should each side of a motorway services area be its own `highway=services` area, and does its `name` carry the direction?
- Should a location photo (overview, entrance, signs) be linked to any OpenStreetMap element: the station area, or the place?
- Should a site's `panoramax=` go on its `man_made=charge_point`, on the station area, or both?
- Tesla's ids exist only in databases and are not visible on site. The community is testing another way to tag them, and the station area rules follow it once it is settled.
