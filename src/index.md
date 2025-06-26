---
toc: false
---

```js
import * as d3 from "d3";
import * as topojson from "topojson-client";
import {legend} from "@observablehq/plot";
```

<div class="hero">
  <h1>Earthquake Analysis</h1>
  <h2>Visualizing earthquake data over time and depth.</h2>
</div>

```js

import {timeline} from "./components/timeline.js";

// Make sure you're using await if you're inside an async function or Observable cell:
const data = await FileAttachment("./static/Updated_Clean_EarthquakeData.csv").csv();

// Assuming your data has a time field as a string (e.g., "2023-04-01T14:32:00"), transform it like this before passing it to Plot:
const dataParsed = data.map(d => ({
  ...d,
  time: new Date(d.time),
  mag: +d.mag,
  depth: +d.depth
}));

// Color scales for magnitude and depth
const magExtent = d3.extent(dataParsed, d => d.mag);
const depthExtent = d3.extent(dataParsed, d => d.depth);

const colorScales = {
  mag: d3.scaleSequential().domain(magExtent).interpolator(d3.interpolateOranges),
  depth: d3.scaleSequential().domain(depthExtent).interpolator(d3.interpolateBlues)
};

let colorMode = "mag"; // remove this line

// Load and convert the TopoJSON land data to GeoJSON
const land = await d3.json("https://cdn.jsdelivr.net/npm/world-atlas@2/land-110m.json")
  .then(world => topojson.feature(world, world.objects.land));

const slider = document.getElementById("rotate");
const globeContainer = document.getElementById("globe");
const color = d3.scaleSequential()
  .domain(d3.extent(dataParsed, d => d.mag)) // use magnitude range
  .interpolator(d3.interpolateOranges);         // orange gradient



```

<!-- time line -->
<div style="margin-bottom: 1rem; display: flex; align-items: center; gap: 1rem;">
  <label for="timeline-slider"><strong>Date:</strong></label>
  <input type="range" id="timeline-slider" min="0" max="100" value="100" style="flex: 1;">
  <span id="timeline-date"></span>
</div>

<style>
#timeline-slider::-webkit-slider-thumb {
  background: #b8dfe0;
}
#timeline-slider::-webkit-slider-runnable-track {
  background: #b8dfe0;
}
#timeline-slider::-moz-range-thumb {
  background: #b8dfe0;
}
#timeline-slider::-moz-range-track {
  background: #b8dfe0;
}
#timeline-slider::-ms-fill-lower,
#timeline-slider::-ms-fill-upper {
  background: #b8dfe0;
}
</style>

<!-- Container for entire visualization -->
<div style="display: flex; align-items: flex-start; gap: 2rem;">

  <!-- World map container -->
  <div id="map-container" style="flex: 1;">
    <div id="world-map-display"></div>
  </div>
    
  <!-- side panel: legend and info box stacked vertically  -->
  <div style="display:flex; flex-direction: column; gap: 1rem; min-width: 240px;">
        <!-- World map projection drop down -->
    <div style="margin-bottom: 1rem;">
      <label for="projection-select"><strong>Map View:</strong></label>
      <select id="projection-select">
        <option value="default">Default (Greenwich-centered)</option>
        <option value="pacific">Pacific-centered</option>
      </select>
      <div style="margin-top: 1rem;">
    <label for="color-mode"><strong>Color by:</strong></label>
    <select id="color-mode">
      <option value="mag">Magnitude</option>
      <option value="depth">Depth</option>
    </select>
    </div>
    <div id="legend-container"></div>
    <div id="legend-depth-container" style="margin-top: 0.5rem;"></div>
    <div id="info-panel" style="
      width: 220px;
      min-height: 100px;
      background: #c5cfaa;
      border: 1px solid #ccc;
      padding: 10px;
      font-size: 13px;
      border-radius: 5px;
    ">
      <strong>Earthquake Info</strong>
      <div id="info-content">Hover over a dot</div>
    </div>
    
  </div>
</div>

```js
// Dimensions
const width = 1000;
const height = 600;

// Projection and path
const projection = d3.geoEqualEarth()
    .translate([width / 2, height / 2])
    .scale(160);

const path = d3.geoPath(projection);

// SVG setup
const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .style("width", "100%")
    .style("height", "auto");

// Group container for zoomable content
const g = svg.append("g");

// Blue ocean background (limited to globe)
g.append("path")
    .datum({ type: "Sphere" })
    .attr("fill", "#b8dfe0")
    .attr("d", path);

// Graticule
g.append("path")
    .datum(d3.geoGraticule10())
    .attr("fill", "none")
    .attr("stroke", "#fff")
    .attr("stroke-opacity", 0.4)
    .attr("d", path);

// Land
g.append("path")
    .datum(land)
    .attr("fill", "#a1b276")
    .attr("stroke", "#a1b276")
    .attr("stroke-width", 0.3)
    .attr("d", path);

// info box
const infoContent = d3.select("#info-content");
// Enable zooming
svg.call(
  d3.zoom()
    .scaleExtent([1, 8])
    .on("zoom", (event) => {
      g.attr("transform", event.transform);
    })
);

// Attach to the page
document.getElementById("world-map-display").innerHTML = "";
document.getElementById("world-map-display").appendChild(svg.node());


// ---------- LEGEND ----------
const legendWidth = 200;
const legendHeight = 10;

const legendSvg = d3.create("svg")
  .attr("width", legendWidth + 40)
  .attr("height", 40);

// Gradient definition
const defs = legendSvg.append("defs");
const linearGradient = defs.append("linearGradient")
  .attr("id", "legend-gradient");

linearGradient.selectAll("stop")
  .data([
    { offset: "0%", color: "#ffffff" },
    { offset: "100%", color: "#ff6600" }
  ])
  .join("stop")
  .attr("offset", d => d.offset)
  .attr("stop-color", d => d.color);

// Gradient rectangle
legendSvg.append("rect")
  .attr("x", 20)
  .attr("y", 20)
  .attr("width", legendWidth)
  .attr("height", legendHeight)
  .style("fill", "url(#legend-gradient)");

// Labels
legendSvg.append("text")
  .attr("x", 20)
  .attr("y", 15)
  .text("Low Magnitude")
  .style("font-size", "10px");

legendSvg.append("text")
  .attr("x", 20 + legendWidth)
  .attr("y", 15)
  .attr("text-anchor", "end")
  .text("High Magnitude")
  .style("font-size", "10px");

// legend placment
document.getElementById("legend-container").innerHTML = "";
document.getElementById("legend-container").appendChild(legendSvg.node());

// ...existing code for magnitude legend...

// ---------- DEPTH LEGEND ----------
const legendDepthSvg = d3.create("svg")
  .attr("width", legendWidth + 40)
  .attr("height", 40);

// Gradient definition for depth
const defsDepth = legendDepthSvg.append("defs");
const linearGradientDepth = defsDepth.append("linearGradient")
  .attr("id", "legend-gradient-depth");

linearGradientDepth.selectAll("stop")
  .data([
    { offset: "0%", color: d3.interpolateBlues(0) },
    { offset: "100%", color: d3.interpolateBlues(1) }
  ])
  .join("stop")
  .attr("offset", d => d.offset)
  .attr("stop-color", d => d.color);

// Gradient rectangle
legendDepthSvg.append("rect")
  .attr("x", 20)
  .attr("y", 20)
  .attr("width", legendWidth)
  .attr("height", legendHeight)
  .style("fill", "url(#legend-gradient-depth)");

// Labels
legendDepthSvg.append("text")
  .attr("x", 20)
  .attr("y", 15)
  .text("Shallow")
  .style("font-size", "10px");

legendDepthSvg.append("text")
  .attr("x", 20 + legendWidth)
  .attr("y", 15)
  .attr("text-anchor", "end")
  .text("Deep")
  .style("font-size", "10px");

// Add depth legend below the magnitude legend
document.getElementById("legend-depth-container").innerHTML = "";
document.getElementById("legend-depth-container").appendChild(legendDepthSvg.node());


// --- Timeline slider setup ---
const slider = document.getElementById("timeline-slider");
const dateLabel = document.getElementById("timeline-date");

// Get min/max dates
const minDate = d3.min(dataParsed, d => d.time);
const maxDate = d3.max(dataParsed, d => d.time);

// Set slider attributes
slider.min = 0;
slider.max = dataParsed.length - 1;
slider.value = slider.max;

// Sort data by time
const dataSorted = dataParsed.slice().sort((a, b) => a.time - b.time);

// Initial map update
updateMap(slider.value);

// Slider event
slider.addEventListener("input", () => {
  updateMap(+slider.value);
});

const colorModeSelect = document.getElementById("color-mode");
colorModeSelect.addEventListener("change", () => {
  document.getElementById("color-mode").value = colorModeSelect.value;
  updateMap(+slider.value);
});

function updateMap(dateIndex) {
  const colorMode = document.getElementById("color-mode").value;
  const selectedDate = dataSorted[dateIndex].time;
  dateLabel.textContent = selectedDate.toLocaleDateString();

  // Filter earthquakes up to selected date
  const filtered = dataSorted.filter(d => d.time <= selectedDate);

  // Update circles
  const circles = g.selectAll("circle")
    .data(filtered, d => d.id || d.time);

  circles.join(
    enter => enter.append("circle")
      .attr("cx", d => projection([d.longitude, d.latitude])[0])
      .attr("cy", d => projection([d.longitude, d.latitude])[1])
      .attr("r", 3)
      .attr("fill", d => colorScales[colorMode](d[colorMode]))
      .attr("fill-opacity", 0.85)
      .on("mouseover", function (event, d) {
        infoContent.html(`
          <strong>Location:</strong> ${d.place}<br>
          <strong>Magnitude:</strong> ${d.mag}<br>
          <strong>Depth:</strong> ${d.depth} km<br>
          <strong>Date:</strong> ${new Date(d.time).toLocaleDateString()}
        `);
      })
      .on("mouseout", function () {
        infoContent.html("Hover over a dot");
      }),
    update => update
      .attr("fill", d => colorScales[colorMode](d[colorMode])),
    exit => exit.remove()
  );
}

```