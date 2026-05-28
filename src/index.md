---
title: USA Formula 1 Circuits
toc: false
---

```js
import * as topojson from "npm:topojson-client";
```

```js
const races = FileAttachment("data/races.csv").csv({typed: true});
const circuits = FileAttachment("data/circuits.csv").csv({typed: true});
```

```js
const us = await d3.json("https://cdn.jsdelivr.net/npm/us-atlas@1/us/10m.json");
```

```js
const scrubberTimeStep = 300
```

```js
const lower48 = {
  type: "FeatureCollection",
  features: topojson
    .feature(us, us.objects.states)
    .features
    .filter((d) => d.id !== "02" && d.id !== "15")
};

const usaCircuitRows = circuits.filter(
  (d) => d.country === "USA" || d.country === "United States");
const usaCircuitIds = new Set(usaCircuitRows.map((d) => d.circuitId));
const usaRaces = races
  .filter((d) => usaCircuitIds.has(d.circuitId))
  .map((d) => ({
    ...d,
    raceDate: d.date instanceof Date ? d.date : new Date(d.date)
  }));

// increase in steps of races
const scrubberDates = Array
  .from(new Set(usaRaces.map((d) => +d.raceDate)))
  .sort(d3.ascending)
  .map((d) => new Date(d));
const dateFormat = d3.utcFormat("%Y %b %-d");
```

```js
const dateIndex = Mutable(scrubberDates.length - 1);

const scrubberControl = (() => {
  const wrap = html`<div class="scrubber-control">
    <button class="scrubber-play" type="button" aria-label="Play timeline">▶</button>
    <div class="scrubber-body">
      <div class="scrubber-header">
        <span class="scrubber-label">Reference date</span>
        <span class="scrubber-value">${dateFormat(scrubberDates[dateIndex.value])}</span>
      </div>
      <input class="scrubber-slider" type="range" min="0" max="${scrubberDates.length - 1}" step="1" value="${dateIndex.value}">
    </div>
  </div>`;

  const button = wrap.querySelector(".scrubber-play");
  const slider = wrap.querySelector(".scrubber-slider");
  const value = wrap.querySelector(".scrubber-value");
  let timer = null;

  const stop = () => {
    if (timer !== null) {
      clearInterval(timer);
      timer = null;
    }
    button.textContent = "▶";
    button.setAttribute("aria-label", "Play timeline");
  };

  const update = (next) => {
    dateIndex.value = next;
    slider.value = next;
    value.textContent = dateFormat(scrubberDates[next]);
  };

  slider.addEventListener("input", () => {
    stop();
    update(+slider.value);
  });

  button.addEventListener("click", () => {
    if (timer !== null) {
      stop();
      return;
    }
    if (+slider.value >= scrubberDates.length - 1) update(0);
    button.textContent = "❚❚";
    button.setAttribute("aria-label", "Pause timeline");
    timer = setInterval(() => {
      const step = +(slider.step || 1);
      const next = Math.min(+slider.value + step, scrubberDates.length - 1);
      update(next);
      if (next >= scrubberDates.length - 1) stop();
    }, scrubberTimeStep);
  });

  invalidation.then(stop);
  return wrap;
})();
```

```js
const selectedDate = scrubberDates[dateIndex];
const windowStart = d3.utcYear.offset(selectedDate, -10);

const globalRacesByCircuit = d3.rollup(
  usaRaces,
  (group) => ({
    raceCount: group.length,
    firstRace: d3.min(group, (d) => d.year) ?? null,
    lastRace: d3.max(group, (d) => d.year) ?? null
  }),
  (d) => d.circuitId  // groupby key
);

const filteredRaces = usaRaces.filter(
  (d) => d.raceDate >= windowStart && d.raceDate <= selectedDate
);

const racesByCircuit = d3.rollup(
  filteredRaces,
  (group) => ({
    raceCount: group.length,
    firstRace: d3.min(group, (d) => d.year) ?? null,
    lastRace: d3.max(group, (d) => d.year) ?? null
  }),
  (d) => d.circuitId  // groupby key
);

const usaCircuits = usaCircuitRows
  .map((d) => {
    const summary = racesByCircuit.get(d.circuitId);
    return {
      ...d,
      raceCount: summary?.raceCount ?? 0,
      firstRace: summary?.firstRace ?? null,
      lastRace: summary?.lastRace ?? null
    };
  })
  .sort((a, b) => d3.descending(a.raceCount, b.raceCount) || d3.ascending(a.name, b.name));

const maxRaceCount = d3.max(globalRacesByCircuit.values(), d => d.raceCount) ?? 0; // global max
console.log(maxRaceCount)
```

# USA Formula 1 circuits

This map shows the U.S. circuits and .

<div class="card scrubber-card">
  ${scrubberControl}
  <div class="muted">
    Showing races from <strong>${dateFormat(windowStart)}</strong> through <strong>${dateFormat(selectedDate)}</strong>.
  </div>
</div>

<div class="map-layout">
  <div class="card">
    ${resize((width) => usCircuitMap(usaCircuits, {width}))}  <!-- For window resizing -->
  </div>
  <div class="sidebar-stack">
    <!-- <div class="card stat-card">
      <h2>Every U.S. circuit in history</h2>
      <span class="big">${usaCircuits.length}</span>
    </div> -->
    <div class="card stat-card">
      <h2>Total U.S. races</h2>
      <span class="big">${d3.sum(usaCircuits, (d) => d.raceCount)}</span>
    </div>
    <div class="card stat-card">
      <h2>Most-used circuit (<strong>${dateFormat(windowStart)} - <strong>${dateFormat(selectedDate)})</h2>
      <span class="big" style="font-size: 1.35rem;">${usaCircuits[0].raceCount > 0 ? usaCircuits[0].name : "None in window"}</span>
    </div>
    <div class="card selected-circuit-card">
      <h2>Historic data of selected circuit</h2>
      <p class="muted">Reserved for a clicked-circuit SVG summary.</p>
    </div>
  </div>
</div>


```js
function usCircuitMap(data, {width}) {
  const height = 500;

  const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .attr("width", width)
    .attr("height", height)
    .attr("style", "max-width: 100%; height: auto;");

  const atlasProjection = d3.geoAlbersUsa()
    .scale(1280)
    .translate([480, 300]);

  const mapProjection = d3.geoIdentity().fitSize([width, height], lower48);
  const path = d3.geoPath(mapProjection);

  const radius = d3.scaleLinear()
    .domain([0, 1, maxRaceCount ** 2])
    .range([3, 6, 80]);

  svg.append("g")
    .selectAll("path")
    .data(lower48.features)
    .join("path")
    .attr("fill", "#f8fafc")
    .attr("stroke", "#94a3b8")
    .attr("stroke-width", 0.75)
    .attr("d", path);

  const pointsG = svg.append("g");

  function renderPoints(data) {
    const pointData = [...data].sort(
      (a, b) => d3.descending(a.raceCount, b.raceCount)
    );

    const points = pointsG
      .selectAll("circle")
      .data(pointData, d => d.circuitId);

    points.join(
      enter => enter.append("circle")
        .attr("cx", d => mapProjection(atlasProjection([d.lng, d.lat]))?.[0])
        .attr("cy", d => mapProjection(atlasProjection([d.lng, d.lat]))?.[1])
        .attr("r", d => radius(d.raceCount ** 2))
        .attr("fill", "#2563eb")
        .attr("stroke", "black")
        .attr("stroke-width", 1.5)
        .attr("cursor", "pointer")
        .style("pointer-events", "all")
        .call(enter => enter.append("title")),

      update => update,

      exit => exit.remove()
    )
    .attr("fill-opacity", d => d.raceCount === 0 ? 0.3 : 0.85)
    .attr("stroke-dasharray", d => d.raceCount === 0 ? "0.5, 1" : "none")
    .select("title")
    .text(d =>
      `${d.name}
      ${d.location}
      Races: ${d.raceCount}
      Years: ${d.firstRace ?? "—"}–${d.lastRace ?? "—"}`
    );

    pointsG.selectAll("circle")
      .data(pointData, d => d.circuitId)
      .transition().duration(scrubberTimeStep - 0.9*scrubberTimeStep).ease(d3.easeSinIn)
        .attr("r", d => radius(d.raceCount ** 2));
  }

  renderPoints(data);

  return svg.node();
}
```



<style>
.scrubber-control {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 0.75rem;
}

.scrubber-play {
  width: 2.75rem;
  height: 2.75rem;
  border: 0;
  border-radius: 999px;
  background: linear-gradient(135deg, #2563eb, #1d4ed8);
  color: white;
  font-size: 1rem;
  font-weight: 700;
  cursor: pointer;
  box-shadow: 0 8px 18px rgba(37, 99, 235, 0.3);
}

.scrubber-play:hover {
  background: linear-gradient(135deg, #3b82f6, #2563eb);
}

.scrubber-body {
  flex: 1;
}

.scrubber-header {
  display: flex;
  justify-content: space-between;
  font-size: 0.95rem;
}

.scrubber-label {
  color: var(--theme-foreground-muted);
}

.scrubber-value {
  font-variant-numeric: tabular-nums;
  font-weight: 600;
}

.scrubber-slider {
  width: 100%;
}

.map-layout {
  display: grid;
  grid-template-columns: minmax(0, 2.1fr) minmax(320px, 0.9fr);
  gap: 0.75rem;
  align-items: start;
}

.sidebar-stack {
  display: grid;
  gap: 0;
}

.stat-card {
  min-height: 60px;
}

.selected-circuit-card {
  min-height: 250px;
}

@media (max-width: 1100px) {
  .map-layout {
    grid-template-columns: 1fr;
  }
}
</style>
