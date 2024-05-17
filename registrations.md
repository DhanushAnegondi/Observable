# Dashboard for Registration User Interface

```js
import * as d3 from "npm:d3";
import {FileAttachment} from "npm:@observablehq/stdlib";

const data = FileAttachment("/data/RUI Registration dataset cleaned-2.csv").csv({typed:true});
display(data);
```

```js
import * as Plot from "npm:@observablehq/plot";
//'top_10_organs' data is available in this structure:
const brands = [
  { name: "lung", value: 25 },
  { name: "liver", value: 14 },
  { name: "pancreas", value: 13 },
  { name: "kidney", value: 13 },
  { name: "bone marrow", value: 7 },
  { name: "Brain", value: 7 },
  { name: "large intestine", value: 6 },
  { name: "fetal development", value: 5 },
  { name: "heart", value: 5 },
  { name: "Blood", value: 5 }
];

const brandsPlot = Plot.plot({
  marginLeft: 90,
  x: { axis: null },
  y: { label: null },
  color: {
    range: ["#000"] // Adjust the bar color if needed
  },
  marks: [
    Plot.barX(brands, {
      x: "value",
      y: "name",
      fill: "#000", // Adjust the bar fill color if needed
      sort: { y: "x", reverse: true, limit: 10 }
    }),
    Plot.text(brands, {
      text: d => d.value, // 
      y: "name",
      x: "value",
      textAnchor: "end",
      dx: -3,
      fill: "white"
    })
  ]
});

display(brandsPlot);

```
```js
async function fetchReferenceOrgans() {
  const url = 'https://apps.humanatlas.io/hra-api/v1/reference-organs';
  try {
    const response = await fetch(url);
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
    const data = await response.json();
    const referenceOrgans = [...new Set(data.map(item => item.label).filter(label => label))];
    return referenceOrgans;
  } catch (e) {
    console.error(`An error occurred while fetching data: ${e}`);
    return [];
  }
}

```
```js
// 
async function createVisualization() {
  const referenceOrgans = await fetchReferenceOrgans();

  

  const registrationRollup = d3.rollups(await data, 
                                         group => group.length, 
                                         d => d.Organ, 
                                         d => d["Dataset Type"]);

  const registrationData = registrationRollup.flatMap(([organ, types]) => 
    types.map(([datasetType, count]) => ({
      organ,
      datasetType,
      count
    }))
  );

  const charty = Plot.plot({
      width: 960,
      marginLeft: 150,
      marginTop: 230,
      grid: true,
      x: {
        axis: "top",
        label: "Organ",
        domain: referenceOrgans, // Use fetched organ labels here
        tickRotate: -90,
        labelOffset: 200,
      },
      y: {
        label: "Dataset Type",
        domain: d3.map(await data, d => d["Dataset Type"]),
      },
      color: {
        type: "linear",
        legend: true,
        domain: [0, d3.max(registrationData, d => d.count)],
        range: ["skyblue", "green"]
      },
      marks: [
        Plot.cell(registrationData, {
          x: "organ",
          y: "datasetType",
          fill: "count",
          inset: 0.5
        }),
        Plot.text(registrationData, {
          x: "organ",
          y: "datasetType",
          text: d => d.count,
          fill: "black"
        })
      ]
  });

  // Display the chart
  display(charty);
}

createVisualization();

```

```js
const registrationStatuses = [
  "Need Author Info", // Represented by 1 in the dataset
  "No reply from author", // Represented by 2 in the dataset
  "Meeting scheduled", // Represented by 3 in the dataset
  "Collecting metadata", // Represented by 4 in the dataset
  "RUI files in-progress", // Represented by 5 in the dataset
  "RUI files exist", // Represented by 6 in the dataset
  "RUI configuration files complete", // Represented by 7 in the dataset
  "Complete & in public EUI", // Represented by 8 in the dataset
  "NOT PARTICIPATING" // Represented by 9 in the dataset
];

async function createVisualization() {
  const referenceOrgans = await fetchReferenceOrgans();
  const normalizedReferenceOrgans = referenceOrgans.map(organ => organ.toLowerCase());

  const filteredData = data.filter(d =>
    d.Organ && normalizedReferenceOrgans.includes(d.Organ.toLowerCase())
  );

  const registrationRollup = d3.rollups(filteredData, 
                                         group => group.length, 
                                         d => d.Organ, 
                                         d => d["Dataset Type"]).flatMap(([organ, types]) => 
    types.map(([datasetType, count]) => ({
      organ,
      datasetType,
      count,
      statusLabel: registrationStatuses[count - 1] // Convert status code to label
    }))
  );

  // Create a color scale for the registration statuses
  const colorScale = d3.scaleOrdinal()
    .domain(registrationStatuses)
    .range(d3.schemeTableau10);

  // Define the plot
  const plot = Plot.plot({
    // Set margins to accommodate labels
    marginLeft: 150,
    marginBottom: 150,
    width: 960,
    height: 600,
    x: {
      label: "Organs",
      domain: filteredData.map(d => d.Organ),
      tickRotate: -45,
      labelOffset: 10,
      padding: 0.1
    },
    y: {
      label: "Registration Status",
      domain: registrationStatuses,
      grid: true
    },
    color: {
      label: "Registration Status",
      type: "categorical",
      domain: registrationStatuses,
      range: colorScale.range(),
    },
    marks: [
      Plot.dot(registrationRollup, { x: "organ", y: "statusLabel", fill: "statusLabel", title: d => `${d.organ}: ${d.statusLabel} in ${d.datasetType}` }),
      
    ],
  });

  display(plot);
}

createVisualization();
```

```js
async function createVisualization() {
  // Fetch the reference organ labels
  const referenceOrgans = await fetchReferenceOrgans();
  const normalizedReferenceOrgans = referenceOrgans.map(organ => organ.toLowerCase());
  const filteredData = data.filter(d =>
    d.Organ && normalizedReferenceOrgans.includes(d.Organ.toLowerCase())
  ).map(d => ({
    ...d,
    'Registration Status Label': registrationStatuses[d['Registration Status'] - 1] || "Unknown"
  }));

  const datasetOrganRollup = d3.rollups(filteredData,
    group => d3.rollup(group, g => g.length, d => d['Registration Status Label']),
    d => d['Dataset Type'],
    d => d.Organ
  ).flatMap(([datasetType, organs]) => {
    return Array.from(organs, ([organ, statuses]) => {
      return Array.from(statuses, ([statusLabel, count]) => ({
        datasetType,
        organ,
        statusLabel,
        count
      }));
    }).flat();
  });
  const colorScale = d3.scaleOrdinal()
    .domain(registrationStatuses)
    .range(d3.schemeTableau10);

  // Define the plot
  const ploty = Plot.plot({
    marginLeft: 150,
    marginBottom: 150,
    width: 960,
    height: 600,
    x: {
      label: "Dataset Types",
      domain: filteredData.map(d => d['Dataset Type']),
      tickRotate: -45,
      labelOffset: 10,
      padding: 0.1
    },
    y: {
      label: "Registration Status",
      domain: registrationStatuses,
      grid: true
    },
    color: {
      label: "Registration Status",
      type: "categorical",
      domain: registrationStatuses,
      range: colorScale.range(),
      },
    marks: [
      Plot.dot(datasetOrganRollup, { 
        x: "datasetType", 
        y: "statusLabel", 
        fill: "statusLabel", 
        title: d => `${d.organ}: ${d.statusLabel} (${d.count})`
      }),
    ],
    });

display(ploty);
}

createVisualization();
```

```js
const registrationRollup = d3.rollups(await data, 
                                       group => group.length, 
                                       d => d.Organ, 
                                       d => d["Dataset Type"]);

const registrationData = registrationRollup.flatMap(([organ, types]) => 
  types.map(([datasetType, count]) => ({
    organ,
    datasetType,
    count
  }))
);

const chartyy = Plot.plot({
    width:960,

  marginLeft: 150,
  marginTop: 230,
  grid: true,
  x: {
    axis: "top",
    label: "Organ",
    domain: d3.map(await data, d => d.Organ), 
    tickRotate: -90, 
    labelOffset: 200, 
  },
  y: {
    label: "Dataset Type",
    domain: d3.map(await data, d => d["Dataset Type"]),
  },
  color: {
    type: "linear",
    legend: true, 
    domain: [0, d3.max(registrationData, d => d.count)],
    range: ["skyblue", "green"] //
  },
  marks: [
    Plot.cell(registrationData, {
      x: "organ",
      y: "datasetType",
      fill: "count",
      inset: 0.5
    }),
    Plot.text(registrationData, {
      x: "organ",
      y: "datasetType",
      text: d => d.count,
      fill: "black"
    })
  ]
});

// Display the chart
display(chartyy);
```

```js

const registrationData = await FileAttachment("RUI Registration Dashboard.csv").csv({ typed: true });

const statusStages = [
  "Need Author Info",
  "No reply from author",
  "Meeting scheduled",
  "Collecting metadata",
  "RUI files in-progress",
  "RUI files exist",
  "RUI configuration files complete",
  "Complete & in public EUI",
  "NOT PARTICIPATING"
];

const statusColors = {
  "Need Author Info": "#e41a1c",
  "No reply from author": "#377eb8",
  "Meeting scheduled": "#4daf4a",
  "Collecting metadata": "#984ea3",
  "RUI files in-progress": "#ff7f00",
  "RUI files exist": "#ffff33",
  "RUI configuration files complete": "#a65628",
  "Complete & in public EUI": "#f781bf",
  "NOT PARTICIPATING": "#999999"
};

// Process data to fit visualization requirements
const processedData = registrationData.map(d => ({
  ...d,
  "Color": statusColors[d["Registration Status"]],
  "Value": statusStages.indexOf(d["Registration Status"]) + 1 // Assign a numeric value based on status
}));

//Create the plot
const plot = Plot.plot({
  marginLeft:220,
  x: {
    axis: "top",
    label: "← decrease · Progression (%) · increase →",
    labelAnchor: "center",
    tickFormat: "+",
    percent: true
  },
  color: {
    scheme: "PiYG", 
    type: "ordinal"
  },
  marks: [
    Plot.barX(processedData, {
      x: "Value",
      y: "Registration Set Name",
      fill: d => d.Value, 
      sort: { y: "x" }
    }),
    Plot.gridX({ stroke: "white", strokeOpacity: 0.5 }),
    ...d3.groups(processedData, d => d.Value > 0).map(([growth, sets]) => [
      Plot.axisY({
        x: 0,
        ticks: sets.map(d => d["Registration Set Name"]),
        tickSize: 0,
        anchor: growth ? "left" : "right"
      }),
      Plot.textX(sets, {
        x: "Value",
        y: "Registration Set Name",
        text: d => d["Registration Status"],
        textAnchor: growth ? "start" : "end",
        dx: growth ? 4 : -4,
      })
    ]),
    Plot.ruleX([0])
  ]
});

display(plot);
```
