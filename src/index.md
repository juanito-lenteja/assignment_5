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
const scrubberTimeStep = 300;
```

```js
const selectedCircuitId = Mutable(null);

function clickedPoint(event, d) {
  
  selectedCircuitId.value = d == null ? null : d.circuitId;
}
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
  .map((d) => new Date(d)
);
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
```

```js
const yearlyRaceWindow = d3.range(
  d3.min(usaRaces, (d) => d.year),
  d3.max(usaRaces, (d) => d.year) + 1
).map((year) => {
  const start = new Date(Date.UTC(year - 10, 0, 1));
  const end = new Date(Date.UTC(year, 11, 31, 23, 59, 59, 999));
  return {
    year,
    raceCount: usaRaces.filter((d) => d.raceDate >= start && d.raceDate <= end).length
  };
});
```

```js
const selectedCircuit = selectedCircuitId == null
  ? null
  : usaCircuitRows.find((d) => d.circuitId === selectedCircuitId);

const selectedHistoricRaces = selectedCircuit == null
  ? []
  : usaRaces
      .filter((d) => d.circuitId === selectedCircuit.circuitId)
      .sort((a, b) => d3.ascending(a.raceDate, b.raceDate));

const selectedCircuitSummary = selectedCircuit == null
  ? null
  : {
      ...selectedCircuit,
      raceCount: selectedHistoricRaces.length,
      firstRace: d3.min(selectedHistoricRaces, (d) => d.year) ?? null,
      lastRace: d3.max(selectedHistoricRaces, (d) => d.year) ?? null,
      firstDate: d3.min(selectedHistoricRaces, (d) => d.raceDate) ?? null,
      lastDate: d3.max(selectedHistoricRaces, (d) => d.raceDate) ?? null
    };
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
      <h2>Total U.S. races (<strong>${dateFormat(windowStart)} - <strong>${dateFormat(selectedDate)})</h2>
      <span class="big">${d3.sum(usaCircuits, (d) => d.raceCount)}</span>
    </div>
    <div class="card stat-card">
      <h2>Most-used circuit (<strong>${dateFormat(windowStart)} - <strong>${dateFormat(selectedDate)})</h2>
      <span class="big" style="font-size: 1.35rem;">${usaCircuits[0].raceCount > 0 ? usaCircuits[0].name : "None in window"}</span>
    </div>
    <div class="card selected-circuit-card">
      <h2>Historic data of selected circuit</h2>
      ${selectedCircuitSummary == null
        ? html`<p class="muted">Click on a circuit to see historical data</p>`
        : resize((width) => historicCircuitSummary(selectedCircuitSummary, {width}))}
    </div>
  </div>
  <br>
</div>


## Non-interactive visualization for comparison 

<div class="card">
  ${resize((width) => Plot.plot(nonInteractivePlot))}
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

  svg.append("text")
    .attr("x", width/3.25)
    .attr("y", 20)
    .attr("font-size", 16)
    .attr("font-weight", 700)
    .attr("fill", "currentColor")
    .text(`Races starting on ${dateFormat(windowStart)} through ${dateFormat(selectedDate)}`)


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

  const pointsG = svg.append("g");  // render before
  const tooltip = svg.append("g")
    .attr("display", "none")
    .attr("pointer-events", "none");

  const tooltipBox = tooltip.append("rect")
    .attr("rx", 6)
    .attr("ry", 6)
    .attr("fill", "#111827")
    .attr("fill-opacity", 0.95)
    .attr("stroke", "#475569")
    .attr("stroke-width", 1);

  const tooltipText = tooltip.append("text")
    .attr("fill", "white")
    .attr("font-size", 12)
    .attr("font-family", "var(--sans-serif)");

  function showTooltip(event, d) {
    const lines = [
      d.name,
      d.location,
      `Races: ${d.raceCount}`,
      `Years: ${d.firstRace ?? "—"}–${d.lastRace ?? "—"}`
    ];

    tooltipText.selectAll("tspan")
      .data(lines)
      .join("tspan")
      .attr("x", 10)
      .attr("dy", (_, i) => i === 0 ? "1.1em" : "1.25em") // adds "title"
      .text((line) => line);

    const {width: boxWidth, height: boxHeight} = tooltipText.node().getBBox();
    tooltipBox
      .attr("width", boxWidth + 20)
      .attr("height", boxHeight + 12);

    const [px, py] = d3.pointer(event, svg.node());
    const tooltipWidth = boxWidth + 20;
    const tooltipHeight = boxHeight + 12;
    const x = Math.max(8, Math.min(width - tooltipWidth - 8, px + 12));
    const y = Math.max(8, Math.min(height - tooltipHeight - 8, py - tooltipHeight - 12));

    tooltip.attr("transform", `translate(${x},${y})`).attr("display", null);
  }

  function hideTooltip() {
    tooltip.attr("display", "none");
  }

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
        .on("mouseenter", showTooltip)
        .on("mousemove", showTooltip)
        .on("mouseleave", hideTooltip)
        .on("click", (event, d) => {
          event.stopPropagation();
          clickedPoint(event, d)
        }),

      update => update,

      exit => exit.remove()
    )
    .attr("fill-opacity", d => d.raceCount === 0 ? 0.3 : 0.85)
    .attr("stroke-dasharray", d => d.raceCount === 0 ? "0.5, 1" : "none");

    pointsG.selectAll("circle")
      .data(pointData, d => d.circuitId)
      .transition().duration(scrubberTimeStep - 0.9*scrubberTimeStep).ease(d3.easeSinIn)
        .attr("r", d => radius(d.raceCount ** 2));
  }

  renderPoints(data);

  svg.on("click", (event, d) => clickedPoint(null, null))

  return svg.node();
}
```

```js
function historicCircuitSummary(circuit, {width}) {
  const height = 242;
  const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .attr("width", width)
    .attr("height", height)
    .attr("style", "max-width: 100%; height: auto;");

  svg.append("text")
    .attr("x", 0)
    .attr("y", 20)
    .attr("font-size", 16)
    .attr("font-weight", 700)
    .attr("fill", "currentColor")
    .text(circuit.name);

  svg.append("text")
    .attr("x", 0)
    .attr("y", 42)
    .attr("font-size", 12)
    .attr("fill", "var(--theme-foreground-muted)")
    .text(`${circuit.location}, ${circuit.country}`);

  const stats = [
    {label: "All-time U.S. races", value: circuit.raceCount},
    {label: "First race", value: circuit.firstRace ?? "—"},
    {label: "Most recent race", value: circuit.lastRace ?? "—"}
  ];

  const cardWidth = Math.max(92, (width - 24) / 3);
  const cardHeight = 84;

  const stat = svg.append("g")
    .attr("transform", "translate(0,70)")
    .selectAll("g")
    .data(stats)
    .join("g")
    .attr("transform", (_, i) => `translate(${i * (cardWidth + 12)},0)`);

  stat.append("rect")
    .attr("width", cardWidth)
    .attr("height", cardHeight)
    .attr("rx", 8)
    .attr("fill", "#0f172a")
    .attr("fill-opacity", 0.08)
    .attr("stroke", "#475569")
    .attr("stroke-opacity", 0.35);

  stat.append("text")
    .attr("x", 12)
    .attr("y", 28)
    .attr("font-size", 11)
    .attr("fill", "var(--theme-foreground-muted)")
    .text((d) => d.label);

  stat.append("text")
    .attr("x", 12)
    .attr("y", 62)
    .attr("font-size", 24)
    .attr("font-weight", 800)
    .attr("fill", "currentColor")
    .text((d) => d.value);

  svg.append("text")
    .attr("x", 0)
    .attr("y", 185)
    .attr("font-size", 12)
    .attr("fill", "var(--theme-foreground-muted)")
    .text(circuit.firstDate && circuit.lastDate
      ? `Historical range: ${dateFormat(circuit.firstDate)} – ${dateFormat(circuit.lastDate)}`
      : "No historical races in the dataset.");

  return svg.node();
}
```


```js
  const nonInteractivePlot = {
    title: "Total U.S. races in trailing 10-year window",
    width,
    height: 320,
    marginLeft: 56,
    marginRight: 24,
    marginBottom: 42,
    style: {
      background: "transparent",
      color: "#e5e7eb"
    },
    x: {
      label: "Year (with 10 year sliding window)",
      tickFormat: "d",
      ticks: 10,
      grid: false,
      tickSize: 6,
      stroke: "#cbd5e1",
      tickStroke: "#cbd5e1",
      labelArrow: false
    },
    y: {
      label: "Total U.S. races",
      grid: true,
      tickSize: 0,
      stroke: "#cbd5e1",
      tickStroke: "#cbd5e1",
      gridStroke: "#64748b",
      gridStrokeOpacity: 0.45,
      labelArrow: false
    },
    marks: [
      Plot.ruleY([0], {stroke: "#cbd5e1"}),
      Plot.lineY(yearlyRaceWindow, {
        x: "year",
        y: "raceCount",
        stroke: "#f97316",
        strokeWidth: 3,
        curve: "linear",
        tip: true
      })
    ]
  }
```


<style>
.scrubber-control {
  display: grid;
  grid-template-columns: auto minmax(0, 1fr);
  align-items: start;
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
  min-width: 0;
}

.scrubber-header {
  display: flex;
  justify-content: space-between;
  font-size: 0.95rem;
  margin-bottom: 0.4rem;
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
  min-height: 266px;
}

@media (max-width: 1100px) {
  .map-layout {
    grid-template-columns: 1fr;
  }
}
</style>
