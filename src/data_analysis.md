---
theme: dashboard
title: Data Analysis
toc: false
---


<!-- Load and transform the data -->
```js
const data = await FileAttachment("./static/Updated_Clean_EarthquakeData.csv").csv();
const dataParsed = data.map(d => ({
  ...d,
  time: new Date(d.time),
  country: d.place.includes(",")
    ? d.place.split(",").pop().trim()
    : d.place.trim()
}));

const land = await d3.json("https://cdn.jsdelivr.net/npm/world-atlas@2/land-110m.json")
  .then(world => topojson.feature(world, world.objects.land));

```

<!-- Imports and consts required -->
```js
import * as d3 from "d3";
import * as Plot from "@observablehq/plot";
import * as topojson from "topojson-client";
import { geoOrthographic, geoPath } from "d3-geo";
import { drag } from "d3-drag";
import {legend} from "@observablehq/plot";
import {timeline} from "./components/timeline.js";
```

<div class="hero">
  <h1>Deep Dive into the Data</h1>
  <h2>Understanding the Earthquake data through various interactive plots</h2>
</div>

<div style="display: flex; align-items: flex-start; gap: 2rem;">
  <div id="scatter-container" style="flex: 1; min-width: 0;">
  <div id="scatter-plot"></div>
  <div style="margin-top: 1rem; width: 400px;">
    <input type="range" id="time-slider" min="0" max="100" value="100" style="width: 100%;">
    <div style="display: flex; justify-content: space-between; font-size: 12px;">
      <span id="start-date-label"></span>
      <span id="end-date-label"></span>
    </div>
    <div id="time-label" style="font-size: 13px; margin-top: 0.5rem;"></div>
    <div id="histogram-plot" style="margin-top: 2rem;"></div> 
  </div>
</div>

<style>
#time-slider::-webkit-slider-thumb {
  background: #b8dfe0;
}
#time-slider::-webkit-slider-runnable-track {
  background: #b8dfe0;
}
#time-slider::-moz-range-thumb {
  background: #b8dfe0;
}
#time-slider::-moz-range-track {
  background: #b8dfe0;
}
#time-slider::-ms-fill-lower,
#time-slider::-ms-fill-upper {
  background: #b8dfe0;
}
</style>


  <!-- Frequency plot (bar/line by country) -->
  <div id="chart-container" style="flex: 1; min-width: 0;">
    <div>
      <select id="top-n">
        <option value="5">Top 5</option>
        <option value="10" selected>Top 10</option>
        <option value="15">Top 15</option>
        <option value="20">Top 20</option>
      </select>
      <label for="lineMetric">Line graph metric:</label>
      <select id="lineMetric">
        <option value="avgMag">Average Magnitude</option>
        <option value="avgDepth">Average Depth</option>
      </select>
    </div>
    <div id="chart"></div>
    <div style="margin-top: 1rem;">
  <div style="display: flex; align-items: center; margin-bottom: 0.5rem;">
    <div style="width: 20px; height: 20px; background-color: #a1b276; margin-right: 8px;"></div>
    <span>Number of Earthquakes</span>
  </div>
    <div style="display: flex; align-items: center; margin-bottom: 0.5rem;">
      <svg width="20" height="20" style="margin-right: 8px;">
        <line x1="0" y1="10" x2="20" y2="10" stroke="#D95F02" stroke-width="2" />
        <circle cx="10" cy="10" r="4" fill="#D95F02" />
      </svg>
      <span>Average Magnitude</span>
    </div>
    <div style="display: flex; align-items: center;">
      <svg width="20" height="20" style="margin-right: 8px;">
        <line x1="0" y1="10" x2="20" y2="10" stroke="steelblue" stroke-width="2" />
        <circle cx="10" cy="10" r="4" fill="steelblue" />
      </svg>
      <span>Average Depth</span>
    </div>
  </div>
  </div>
</div>

```js
document.getElementById("scatter-plot").innerHTML = "";

// Clean and filter the data
const cleanData = dataParsed.filter(d =>
  d.mag !== null && !isNaN(d.mag) &&
  d.depth !== null && !isNaN(d.depth)
);

// Sort the data by time
const sortedData = cleanData.sort((a, b) => new Date(a.time) - new Date(b.time));

// Time and scale setup
const timeExtent = d3.extent(sortedData, d => new Date(d.time));
const magExtent = d3.extent(cleanData, d => +d.mag);
const depthExtent = d3.extent(cleanData, d => +d.depth);

const formatDate = d3.timeFormat("%Y-%m-%d");

// Slider and labels
const slider = document.getElementById("time-slider");
const timeLabel = document.getElementById("time-label");
const startDateLabel = document.getElementById("start-date-label");
const endDateLabel = document.getElementById("end-date-label");

// Scales for converting slider <-> time
const timeScale = d3.scaleTime()
  .domain(timeExtent)
  .range([0, 100]);

const inverseTimeScale = d3.scaleLinear()
  .domain([0, 100])
  .range(timeExtent);

// Set initial labels
startDateLabel.textContent = formatDate(timeExtent[0]);
endDateLabel.textContent = formatDate(timeExtent[1]);

// Initial render (end of time range)
updateScatterPlot(timeExtent[1]);
timeLabel.textContent = `Showing earthquakes up to: ${formatDate(timeExtent[1])}`;

// Event listener
slider.addEventListener("input", () => {
  const sliderDate = inverseTimeScale(+slider.value);
  timeLabel.textContent = `Showing earthquakes up to: ${formatDate(sliderDate)}`;
  updateScatterPlot(sliderDate);
});

// Scatter plot function
function updateScatterPlot(dateCutoff) {
  const filtered = sortedData.filter(d => new Date(d.time) <= dateCutoff);

  document.getElementById("scatter-plot").innerHTML = "";
  document.getElementById("scatter-plot").appendChild(
    Plot.plot({
      width: 500,
      height: 280,
      x: {
        label: "Depth (km)",
        domain: depthExtent,
        nice: false,
        type: "linear",
        labelFontSize: 16
      },
      y: {
        label: "Magnitude",
        domain: magExtent,
        nice: false,
        type: "linear",
        labelFontSize: 16 
      },
      marks: [
        Plot.dot(filtered, {
          x: d => +d.depth,
          y: d => +d.mag,
          r: 2,
          fill: "#D95F02",
          fillOpacity: 0.7
        })
      ]
    })
  );
}

```


```js
// bar chart -----------------------------
function prepareData(n) {
  const byCountry = d3.rollups(
    dataParsed,
    v => ({
      count: v.length,
      avgMag: d3.mean(v, d => d.mag),
      avgDepth: d3.mean(v, d => d.depth)
    }),
    d => d.country
  ).filter(d => d[0]) //filters empty/null countries;
  return byCountry
    .sort((a, b) => d3.descending(a[1].count, b[1].count))
    .slice(0, n)
    .map(([country, stats]) => ({
      country,
      count: stats.count,
      avgMag: stats.avgMag,
      avgDepth: stats.avgDepth
    }));
}

function drawChart(n, lineMetric = "avgMag") {
  const chartData = prepareData(n); 

d3.select("#chart").html(""); // Clear old chart
  d3.select("#top-n").on("change", function() {
    const n = +this.value;
    const metric = d3.select("#lineMetric").property("value");
    drawChart(n, metric);
  });

  d3.select("#lineMetric").on("change", function() {
    const n = +d3.select("#top-n").property("value");
    const metric = this.value;
    drawChart(n, metric);
  });

  d3.select("#line-label").text(lineMetric === "avgMag" ? "Average Magnitude" : "Average Depth");
  d3.select("#line-label").text(lineMetric === "avgMag" ? "Average Magnitude" : "Average Depth");
  const margin = { top: 30, right: 50, bottom: 110, left: 50 },
        width = 550 - margin.left - margin.right,
        height = 400 - margin.top - margin.bottom;

  const svg = d3.select("#chart")
    .append("svg")
    .attr("width", width + margin.left + margin.right)
    .attr("height", height + margin.top + margin.bottom)
    .append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`);

  const x = d3.scaleBand()
    .domain(chartData.map(d => d.country))
    .range([0, width])
    .padding(0.2);

  const yLeft = d3.scaleLinear()
    .domain([0, d3.max(chartData, d => d.count)]).nice()
    .range([height, 0]);

  const yRight = d3.scaleLinear()
    .domain([0, d3.max(chartData, d => d[lineMetric])]).nice()
    .range([height, 0]);

  const metricLabels = {
  avgMag: "Average Magnitude",
  avgDepth: "Average Depth [km]"
  };


  // Axes
  svg.append("g")
    .call(d3.axisLeft(yLeft));

  svg.append("g")
    .attr("transform", `translate(${width},0)`)
    .call(d3.axisRight(yRight));

  svg.append("g")
    .attr("transform", `translate(0,${height})`)
    .call(d3.axisBottom(x))
    .selectAll("text")
    .attr("transform", "rotate(-45)") // rotate the text
    .style("text-anchor", "end")      // align the text properly
    .attr("dx", "-0.8em")             // tweak horizontal spacing
    .attr("dy", "0.15em");            // tweak vertical spacing

  // x-axis title
  svg.append("text")
  .attr("text-anchor", "middle")
  .attr("x", width / 2)
  .attr("y", height + margin.bottom - 20) // Slightly below the axis
  .text("Countries");

  // left y-axis title
  svg.append("text")
  .attr("text-anchor", "middle")
  .attr("transform", `rotate(-90)`)
  .attr("x", -height / 2)
  .attr("y", -margin.left + 15) // Adjust this if needed
  .text("Number of Earthquakes");

  // right y-axit title
  svg.append("text")
  .attr("text-anchor", "middle")
  .attr("transform", `rotate(-90)`)
  .attr("x", -height / 2)
  .attr("y", width + margin.right - 10) // Adjust this based on your layout
  .text(metricLabels[lineMetric]);


  // Bars (left y-axis)
  svg.selectAll(".bar")
    .data(chartData)
    .enter()
    .append("rect")
    .attr("x", d => x(d.country))
    .attr("y", d => yLeft(d.count))
    .attr("width", x.bandwidth())
    .attr("height", d => height - yLeft(d.count))
    .attr("fill", "#a1b276"); // colour bar chart

  // Choose color based on metric
const lineColor = lineMetric === "avgMag" ? "#D95F02" : "steelblue";

// Line (right y-axis)
const line = d3.line()
  .x(d => x(d.country) + x.bandwidth() / 2)
  .y(d => yRight(d[lineMetric]));

svg.append("path")
  .datum(chartData)
  .attr("fill", "none")
  .attr("stroke", lineColor)
  .attr("stroke-width", 2)
  .attr("d", line);

// Circles on line
svg.selectAll(".dot")
  .data(chartData)
  .enter()
  .append("circle")
  .attr("cx", d => x(d.country) + x.bandwidth() / 2)
  .attr("cy", d => yRight(d[lineMetric]))
  .attr("r", 4)
  .attr("fill", lineColor);

}

const initialN = +d3.select("#top-n").property("value");
  drawChart(initialN); // Use actual selected value

  d3.select("#top-n").on("change", function() {
  const n = +this.value;
  drawChart(n); // updates when selection changes
  });


```


```js
// interactive time histogram
function updateHistogram(selectedDate) {
  // Group earthquakes by date (format: YYYY-MM-DD)
  const dateCounts = d3.rollups(
    sortedData,
    v => v.length,
    d => formatDate(new Date(d.time))
  ).sort((a, b) => d3.ascending(a[0], b[0]));

  const dates = dateCounts.map(d => d[0]);
  const counts = dateCounts.map(d => d[1]);

  // Find the index of the selected date (or closest previous date)
  const selectedDateStr = formatDate(selectedDate);
  const selectedIndex = dates.findIndex(d => d === selectedDateStr);

  // Render histogram
  document.getElementById("histogram-plot").innerHTML = "";
  const width = 400, height = 150, margin = {top: 20, right: 20, bottom: 60, left: 60};

  const x = d3.scaleBand()
    .domain(dates)
    .range([margin.left, width - margin.right])
    .padding(0.1);

  const y = d3.scaleLinear()
    .domain([0, d3.max(counts)]).nice()
    .range([height - margin.bottom, margin.top]);

  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height);

  // Bars
  svg.append("g")
    .selectAll("rect")
    .data(dateCounts)
    .join("rect")
      .attr("x", d => x(d[0]))
      .attr("y", d => y(d[1]))
      .attr("width", x.bandwidth())
      .attr("height", d => y(0) - y(d[1]))
      .attr("fill", (d, i) => i === selectedIndex ? "#3a6ba1" : "#b8dfe0");

  // X axis (show fewer ticks for readability)
  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(x).tickValues(dates.filter((d, i) => i % Math.ceil(dates.length / 10) === 0)))
    .selectAll("text")
    .attr("transform", "rotate(-45)")
    .style("text-anchor", "end");

  // Y axis
  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).ticks(4));

  // Axis labels
  svg.append("text")
    .attr("x", width / 2)
    .attr("y", height)
    .attr("text-anchor", "middle")
    .attr("font-size", 12)
    .text("Date");

  svg.append("text")
    .attr("transform", "rotate(-90)")
    .attr("x", -height / 2)
    .attr("y", 12)
    .attr("text-anchor", "middle")
    .attr("font-size", 12)
    .text("Earthquake Count");

  document.getElementById("histogram-plot").appendChild(svg.node());
}

// Initial render (end of time range)
updateScatterPlot(timeExtent[1]);
updateHistogram(timeExtent[1]);
timeLabel.textContent = `Showing earthquakes up to: ${formatDate(timeExtent[1])}`;

// Event listener
slider.addEventListener("input", () => {
  const sliderDate = inverseTimeScale(+slider.value);
  timeLabel.textContent = `Showing earthquakes up to: ${formatDate(sliderDate)}`;
  updateScatterPlot(sliderDate);
  updateHistogram(sliderDate);
});
```
```js
// Debug output
// console.log(data); // Uncomment for debugging purposes
data.forEach(quake => {
  if (quake.place) {
    console.log(quake.place);
  } else {
    console.warn("Missing place information for earthquake:", quake);
  }
});
```
