<script lang="ts">
import { getContext } from "svelte";

import {
  CddaData,
  getVehiclePartIdAndVariant,
  itemGroupFromVehicle,
  showProbability,
  singularName,
} from "../data";
import LimitedList from "../LimitedList.svelte";

import type { Vehicle } from "../types";
import ItemSymbol from "./item/ItemSymbol.svelte";
import ThingLink from "./ThingLink.svelte";

export let item: Vehicle;

const data = getContext<CddaData>("data");

let minX = Infinity;
let maxX = -Infinity;
let minY = Infinity;
let maxY = -Infinity;
for (const part of item.parts) {
  if (part.x < minX) minX = part.x;
  if (part.x > maxX) maxX = part.x;
  if (part.y < minY) minY = part.y;
  if (part.y > maxY) maxY = part.y;
}

const standardSymbols = {
  cover: "^",
  cross: "c",
  horizontal: "h",
  horizontal_2: "=",
  vertical: "j",
  vertical_2: "H",
  ne: "u",
  nw: "y",
  se: "n",
  sw: "b",
};

const zOrder = {
  on_roof: 9,
  on_cargo: 8,
  center: 7,
  under: 6,
  structure: 5,
  engine_block: 4,
  on_battery_mount: 3,
  fuel_source: 2,
  roof: -1,
  armor: -2,
};
const LINE = {
  XOXO: "│",
  OXOX: "─",
  XXOO: "└",
  OXXO: "┌",
  OOXX: "┐",
  XOOX: "┘",
  XXXO: "├",
  XXOX: "┴",
  XOXX: "┤",
  OXXX: "┬",
  XXXX: "┼",
};

const specialSymbol = (symbol: string): string => {
  switch (symbol) {
    case "j":
      return LINE.XOXO;
    case "h":
      return LINE.OXOX;
    case "c":
      return LINE.XXXX;
    case "y":
      return LINE.OXXO;
    case "u":
      return LINE.OOXX;
    case "n":
      return LINE.XOOX;
    case "b":
      return LINE.XXOO;
    default:
      return symbol;
  }
};

const symbolForVehiclePartVariant = (
  partId: string,
  variant: string
): string => {
  const vehiclePart = data.byId("vehicle_part", partId);
  const symbol = vehiclePart.symbol ?? "=";
  const symbols = {
    ...(vehiclePart.standard_symbols ? standardSymbols : {}),
    ...vehiclePart.symbols,
  };
  return specialSymbol(symbols[variant] ?? symbol);
};

const colorForVehiclePart = (partId: string) => {
  const vehiclePart = data.byId("vehicle_part", partId);
  const color = vehiclePart.color;
  return color;
};

type NormalizedPart = { partId: string; variant: string; fuel?: string };
type NormalizedPartList = {
  x: number;
  y: number;
  parts: NormalizedPart[];
};
const normalizedParts: NormalizedPartList[] = item.parts.map((part) => {
  const parts = (
    part.part
      ? [{ part: part.part, fuel: part.fuel }]
      : part.parts?.map((part) => (typeof part === "string" ? { part } : part))
  ).map(({ part, fuel }) => {
    const [partId, variant] = getVehiclePartIdAndVariant(data, part);
    return {
      partId,
      variant,
      fuel,
    };
  });
  return {
    x: part.x,
    y: part.y,
    parts,
  };
});

const zForPart = (partId: string): number => {
  const vehiclePart = data.byId("vehicle_part", partId);
  const location = vehiclePart.location;
  const z = zOrder[location] ?? 0;
  return z;
};

const grid: (NormalizedPart | undefined)[][] = [];
for (let x = maxX; x >= minX; x--) {
  const row: (NormalizedPart | undefined)[] = [];
  for (let y = minY; y <= maxY; y++) {
    const parts = normalizedParts
      .filter((p) => p.x === x && p.y === y)
      .flatMap((p) => p.parts);
    if (!parts.length) {
      row.push(undefined);
      continue;
    }
    let topPart = parts[0];
    let topZ = zForPart(topPart.partId);
    for (const part of parts) {
      const z = zForPart(part.partId);
      if (z >= 0 && z > topZ) {
        topZ = z;
        topPart = part;
      }
    }
    row.push(topPart);
  }
  grid.push(row);
}

const items = data.flattenItemGroup(itemGroupFromVehicle(item));
items.sort((a, b) => b.prob - a.prob);

function groupBy<T>(things: T[], groupsFor: (x: T) => string[]) {
  const map = new Map<string, T[]>();
  for (const thing of things) {
    const groups = groupsFor(thing);
    for (const group of groups) {
      if (!map.has(group)) map.set(group, []);
      map.get(group).push(thing);
    }
  }
  return map;
}
const parts = normalizedParts.flatMap((np) => np.parts);
const partsGrouped = groupBy(parts, (p) => [p.partId]);
const partsCounted = [...partsGrouped.entries()].map(([id, list]) => ({
  id,
  count: list.length,
}));
partsCounted.sort((a, b) => {
  if (a.count === b.count)
    return singularName(data.byId("vehicle_part", a.id)).localeCompare(
      singularName(data.byId("vehicle_part", b.id))
    );
  else return b.count - a.count;
});
</script>

<h1>{singularName(item)}</h1>

<section>
  <pre
    style="font-family: Unifont, monospace; line-height: 1">
    {#each grid as row}
    {#each row as part}{#if part}<span title={`${part.partId}_${part.variant}`} class={`c_${colorForVehiclePart(part.partId)}`}>{symbolForVehiclePartVariant(part.partId, part.variant)}</span>{:else}{" "}{/if}{/each}{'\n'}
    {/each}
  </pre>
</section>

{#if partsCounted.length}
  <section>
    <h1>Parts</h1>
    <LimitedList items={partsCounted} let:item={{ id, count }}>
      <ThingLink {id} type="vehicle_part" /> ({count})
    </LimitedList>
  </section>
{/if}

{#if items.length}
  <section>
    <h1>Items</h1>
    <LimitedList {items} let:item>
      <ItemSymbol item={data.byId("item", item.id)} />
      <ThingLink id={item.id} type="item" />
      ({showProbability(item.prob)})
    </LimitedList>
  </section>
{/if}
