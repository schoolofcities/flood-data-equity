<script>
  import {onMount} from "svelte";
  import maplibregl from "maplibre-gl";
  import {ScaleControl, NavigationControl} from "maplibre-gl";
  import conservationAuthority from "../data/gta-conservation-authority.geo.json";
  import municipalities from "../data/gta-municipalities.geo.json";
  import uppertier from "../data/gta-upper-tier-municipalities.geo.json";
  import * as turf from "@turf/turf"; // this is for fitting the map boundary to GTA municipalities
  import lookupTable from "../data/lookupTable.json";
  import Select from "svelte-select";
  import Papa from "papaparse";

  const municipalCsv =
    "https://docs.google.com/spreadsheets/d/e/2PACX-1vQT7hsW3C1bVjp8xP8d-3HtXAMp8tQOUYOCxABymKbuOQP4TWkEDAB3wut7g1tO5Mw527PHFm_tn-dz/pub?gid=0&single=true&output=csv";
  const regionCsv =
    "https://docs.google.com/spreadsheets/d/e/2PACX-1vQT7hsW3C1bVjp8xP8d-3HtXAMp8tQOUYOCxABymKbuOQP4TWkEDAB3wut7g1tO5Mw527PHFm_tn-dz/pub?gid=193533627&single=true&output=csv";
  const consAuthCsv =
    "https://docs.google.com/spreadsheets/d/e/2PACX-1vQT7hsW3C1bVjp8xP8d-3HtXAMp8tQOUYOCxABymKbuOQP4TWkEDAB3wut7g1tO5Mw527PHFm_tn-dz/pub?gid=898330427&single=true&output=csv";
  // loads a list of unique jurisdictions (i.e. municipality, region, conservation authority)
  let popupContent = false;

  let jurisdictionInfo;
  let map;

  let conservationFilter;
  let municipalFilter;
  let regionalFilter;
  let selectedCon = "";
  let selectedReg = "";
  let selectedMun = "";

  let municipalData = []; // storing municipal data and links
  let regionalData = []; // storing regional data and links
  let conservationData = []; // storing conservation authority data and links
  let selectedJurisdiction = []; //storing selected jurisdiction data and links.
  let dataLoaded = false

  var cityList = [
    "BRAMPTON",
    "BURLINGTON",
    "MARKHAM",
    "MISSISSAUGA",
    "PICKERING",
    "RICHMOND HILL",
    "TORONTO",
    "VAUGHAN",
  ];

  // ============================functions=============================================

  // this is for handling the imported data and joining with the GIS layer
  async function handleCsvData(data, header) {
    // Assuming 'MUNID' is the common key and find the index of regional and municipal layer count
    const munCountIndex = header.indexOf("MUN_LAYER");
    try {
      municipalities.features.forEach((feature) => {
        // Find matching record in CSV data
        const matchingRecord = data.find(
          (obj) => obj.CSDUID.toString() === feature.properties.CSDUID,
        );
        // If a match is found, update GeoJSON properties with CSV data
        if (matchingRecord) {
          feature.properties[header[munCountIndex]] = parseInt(
            matchingRecord.MUN_LAYER,
          );
        }
      });

      // this is to update those with no MUN_LAYER values to 0
      municipalities.features.forEach((feature) => {
        // Find matching record in CSV data
        if (feature.properties[header[munCountIndex]] === undefined) {
          feature.properties[header[munCountIndex]] = 0;
        }
      });

      return true;
      // Now municipalities GeoJSON features have additional properties from the CSV data
    } catch (error) {
      console.error("Error fetching or processing CSV:", error);
      return false;
    }
  }

  // load the google sheet csv data
  async function processCsv(csvLink) {
    const response = await fetch(csvLink);
    const csvData = await response.text();
    const result = await new Promise((resolve) => {
      Papa.parse(csvData, {
        complete: (result) => resolve(result),
        header: true,
        dynamicTyping: true,
        skipEmptyLines: true,
      });
    });

    return result;
  }
  // load the data
  onMount(async () => {
    municipalData = await processCsv(municipalCsv);
    regionalData = await processCsv(regionCsv);
    conservationData = await processCsv(consAuthCsv);

    municipalData = municipalData.data;
    regionalData = regionalData.data;
    conservationData = conservationData.data;

    console.log(municipalData)
    console.log(regionalData)
    console.log(conservationData)
  });

  // loading lookuptable to create unique jurisdiction lists.(i.e all the names of the municipalities)
  function jurisDictionListing(jurisdiction) {
    // jurisdiction is the level of government we want to pull from. 
    // inputs includes:  MUNICIPAL_NAME, REGIONAL_NAME, CONSERVATION_NAME
    let filteredList = [];
    for (let i = 0; i < lookupTable.length; i++) {
      if (!filteredList.includes(lookupTable[i][jurisdiction])) {
        if (!lookupTable[i][jurisdiction] == "") {
          filteredList.push(lookupTable[i][jurisdiction]);
        }
      }
    }
    return filteredList;
  }

  // this is for filtering a list of relevant jurisdictions to the jurisdiction that users select in their dropdown list
  function govFiltering(lookupTable, filtergov, government, jurisdiction) {
    // lookupTable is the lookupTable json file
    // filtergov is the level of government of the selected jurisdiction, input includes MUNICIPAL_NAME, REGIONAL_NAME, CONSERVATION_NAME
    // government is the information of the level of government we want to pull from, input includes  MUNICIPAL_NAME, REGIONAL_NAME, CONSERVATION_NAME
    // jurisdiction is selected value from the drop down list, input can be: Regional Municipality of York, City of Toronto, Town of Ajax, and Toronto and Regional Conservation Authority

    let list = [];
    for (let i = 0; i < lookupTable.length; i++) {
      if (lookupTable[i][filtergov] === jurisdiction) {
        if (!list.includes(lookupTable[i][government])) {
          list.push(lookupTable[i][government]);
        }
      }
    }
    return list;
  }

  function dataFiltering(csvData, jurisdictionList) {
    // csvData is the municipal, regional, and conservation authority data csvData contains all the data for each respective level of government, 
    // we don't need all that data, so we need this function to filter the ones we want. 

    // jurisdictionList is a list of relevant jurisdictions that we got from the govFiltering() function. 

    // we want to loop through each jurisdiction that is within the jurisdiction list, for each jurisdiction,
    // add the fitting rows into the matching list
    let matchingList = [];
    // loop though the jurisdiction list
    for (let j = 0; j < jurisdictionList.length; j++) {
      // loop through all the csvData to find matching jurisdictions
      for (let i = 0; i < csvData.length; i++) {
        //remember to convert lookuptable value to upper case, to match with the official name in csvdata
        if (csvData[i].OFFICIAL_NAME === jurisdictionList[j].toUpperCase()) {
          matchingList.push(csvData[i]);
        }
      }
    }
    console.log(matchingList)
    return matchingList;
  }

  function governmentList(jurisdiction) {
    // jurisdiction is the selected value from the drop down menu, that value will come from the selectMunDropdown, selectRegDropdown, selectConDropdown
    // the function will return two lists, if jurisdiction is regional, it will return municipal and conservation, if jurisdiction is conservation, 
    // it will return municipal and regional, if jurisdiction is municipal, then it returns conservation and regional    
    let region = [];
    let conservation = [];
    let municipal = [];

    // the input jurisdicton can be a local, regional, or conservation authority level jurisdicaiton,
    // so we have identify which group it belongs, because the returned list will be different
    if (jurisdiction.startsWith("Regional")) {
      // if this is a regional municipality
      municipal = govFiltering(
        lookupTable,
        "REGION_NAME",
        "MUNICIPAL_NAME",
        jurisdiction,
      );
      conservation = govFiltering(
        lookupTable,
        "REGION_NAME",
        "CONSERVATION_NAME",
        jurisdiction,
      );
      // this is the list of data links within the selected jurisdiction.
      selectedJurisdiction = dataFiltering(regionalData, [jurisdiction]);
      // these are the combined list of other jurisdictions the selected jurisdiction overlaps
      municipalFilter = dataFiltering(municipalData, municipal);
      conservationFilter = dataFiltering(conservationData, conservation);
      regionalFilter = [] // have this here so that it clears out the municipalFilter values from previous selection
      return [selectedJurisdiction, municipalFilter, conservationFilter];
    } else if (jurisdiction.endsWith("Authority")) {
      // if this is a conservation authority
      region = govFiltering(
        lookupTable,
        "CONSERVATION_NAME",
        "REGION_NAME",
        jurisdiction,
      );
      municipal = govFiltering(
        lookupTable,
        "CONSERVATION_NAME",
        "MUNICIPAL_NAME",
        jurisdiction,
      );
      // this is the list of data links within the published by the selected jurisdiction.
      selectedJurisdiction = dataFiltering(conservationData, [jurisdiction]);
      // these are the combined list of other jurisdictions the selected one overlaps
      municipalFilter = dataFiltering(municipalData, municipal);
      regionalFilter = dataFiltering(regionalData, region);
      conservationFilter = [] // have this here so that it clears out the municipalFilter values from previous selection
      return [selectedJurisdiction, municipalFilter, regionalFilter];
    } else {
      municipal = [];
      // if this is a local municipality
      region = govFiltering(
        lookupTable,
        "MUNICIPAL_NAME",
        "REGION_NAME",
        jurisdiction,
      );
      conservation = govFiltering(
        lookupTable,
        "MUNICIPAL_NAME",
        "CONSERVATION_NAME",
        jurisdiction,
      );

      // this is the list of data links within the published by the selected jurisdiction.
      selectedJurisdiction = dataFiltering(municipalData, [jurisdiction]);
      // these are the combined list of other jurisdictions the selected one overlaps
      regionalFilter = dataFiltering(regionalData, region);
      conservationFilter = dataFiltering(conservationData, conservation);
      municipalFilter = [] // have this here so that it clears out the municipalFilter values from previous selection
      return [selectedJurisdiction, regionalFilter, conservationFilter];
    }
  }
  // the dropdown for municipalities
  function selectMunDropdown() {
    selectedJurisdiction = [];
    conservationFilter = [];
    regionalFilter = [];

    // get the filtered results
    jurisdictionInfo = selectedMun;
    console.log(jurisdictionInfo);
    //filtering results
    selectedJurisdiction = governmentList(jurisdictionInfo)[0];
    regionalFilter = governmentList(jurisdictionInfo)[1];
    conservationFilter = governmentList(jurisdictionInfo)[2];

    selectedCon = "";
    selectedReg = "";

    let clean = jurisdictionInfo
      .replace("City of ", "")
      .replace("Town of ", "")
      .replace("Regional Municipality of ", "");

    popupContent = true;

    // this is to filter the map so that it shows red boundary on the jurisdiction that people select
    map.setFilter("uppertier-border-highlight", ["==", ["get", "CDNAME"], clean]);
    map.setFilter("municipalities-border-highlight", ["==", ["get", "CSDNAME"], clean]);
    map.setFilter("conservationAuthority-border-highlight", [
      "==",
      ["get", "LEGAL_NAME"],
      jurisdictionInfo,
    ]);
  }
  // the dropdown for regional
  function selectRegDropdown() {
    selectedJurisdiction = [];
    municipalFilter = [];
    regionalFilter = [];

    // get the filtered results
    jurisdictionInfo = selectedReg;
    console.log(jurisdictionInfo);
    selectedJurisdiction = governmentList(jurisdictionInfo)[0];
    municipalFilter = governmentList(jurisdictionInfo)[1];
    conservationFilter = governmentList(jurisdictionInfo)[2];

    selectedMun = "";
    selectedCon = "";

    let clean = jurisdictionInfo
      .replace("City of ", "")
      .replace("Town of ", "")
      .replace("Regional Municipality of ", "");

    popupContent = true;

    // this is to filter the map so that it shows red boundary on the jurisdiction that people select
    map.setFilter("uppertier-border-highlight", ["==", ["get", "CDNAME"], clean]);
    map.setFilter("municipalities-border-highlight", ["==", ["get", "CSDNAME"], clean]);
    map.setFilter("conservationAuthority-border-highlight", [
      "==",
      ["get", "LEGAL_NAME"],
      jurisdictionInfo,
    ]);
  }

  // the dropdown for conservation authority
  function selectConDropdown() {
    selectedJurisdiction = [];
    municipalFilter = [];
    conservationFilter = [];

    // get the filtered results
    jurisdictionInfo = selectedCon;
    selectedJurisdiction = governmentList(jurisdictionInfo)[0];
    municipalFilter = governmentList(jurisdictionInfo)[1];
    regionalFilter = governmentList(jurisdictionInfo)[1];

    selectedMun = "";
    selectedReg = "";

    let clean = jurisdictionInfo
      .replace("City of ", "")
      .replace("Town of ", "")
      .replace("Regional Municipality of ", "");

    popupContent = true;

    // this is to filter the map so that it shows red boundary on the jurisdiction that people select
    map.setFilter("uppertier-border-highlight", ["==", ["get", "CDNAME"], clean]);
    map.setFilter("municipalities-border-highlight", ["==", ["get", "CSDNAME"], clean]);
    map.setFilter("conservationAuthority-border-highlight", [
      "==",
      ["get", "LEGAL_NAME"],
      jurisdictionInfo,
    ]);
  }
  // ============================Loading Data and Maps=============================================

  // obtain a data for each level of government
  let conservationList = jurisDictionListing("CONSERVATION_NAME");
  let regionList = jurisDictionListing("REGION_NAME");
  let municipalList = jurisDictionListing("MUNICIPAL_NAME");

  

  onMount(async () => {
    // only load the maps when the google sheet data is loaded
    // read the csvfile from google sheets

    console.log(dataLoaded)
    
    const csv = await processCsv(municipalCsv);
    dataLoaded = handleCsvData(csv.data, csv.meta.fields);
    
    if (dataLoaded) {
      map = new maplibregl.Map({
        container: "map",
        style: "https://basemaps.cartocdn.com/gl/positron-gl-style/style.json", //'https://basemaps.cartocdn.com/gl/positron-gl-style/style.json',
        center: [-79.0, 44.1], // starting position
        zoom: 8, // starting zoom;
        minZoom: 2,
        maxZoom: 17,
        projection: "globe",
        scrollZoom: true,
        attributionControl: false,
      });
      // Adding scale bar to the map
      let scale = new maplibregl.ScaleControl({
        maxWidth: 100,
        unit: "metric",
      });
      // Adding additional layers from geojson
      map.on("load", function () {
        map.addControl(new maplibregl.NavigationControl(), "top-right");
        map.addControl(scale, "bottom-right");
        const layers = map.getStyle().layers;

        // Find the index of the first symbol layer in the map style.
        let firstSymbolId;
        for (const layer of layers) {
          if (layer.type === "symbol") {
            firstSymbolId = layer.id;
            break;
          }
        }
        map.addSource("conservationAuthority", {
          type: "geojson",
          data: conservationAuthority,
        });
        map.addSource("municipalities", {
          type: "geojson",
          data: municipalities,
        });
        map.addSource("uppertier", {
          type: "geojson",
          data: uppertier,
        });
        map.addLayer({
          id: "conservationAuthority-fill",
          type: "fill",
          source: "conservationAuthority",
          layout: {},
          paint: {
            "fill-color": "#FFFFFF",
            "fill-opacity": 0,
          },
        });
        map.addLayer({
          id: "municipalities",
          type: "fill",
          source: "municipalities",
          layout: {},
          paint: {
            "fill-color": [
              "step",
              ["get", "MUN_LAYER"], // Property in your GeoJSON data containing the values
              "white",
              0,
              "grey",
              1, // 1 or lower
              "#a9d6e5",
              3, // 3
              "#89c2d9",
              4, // 4-7
              "#2c7da0",
              7, // 7-10
              "#2a6f97",
              10,
              "#013a63",
            ],
            "fill-opacity": 0.7,
          },
        });
        map.addLayer({
          id: "municipalities-border",
          type: "line",
          source: "municipalities",
          layout: {},
          paint: {
            "line-color": "#000000", // Border color
            "line-width": 1, // Border width
          },
        });
        map.addLayer({
          id: "uppertier-border",
          type: "line",
          source: "uppertier",
          layout: {},
          paint: {
            "line-color": "grey", // Border color
            "line-width": 3, // Border width
          },
        });

        // Fit the map to the bounds of the polygon
        var bbox = turf.bbox(municipalities); // Using Turf.js to calculate the bounding box
        map.fitBounds(bbox, { padding: 20 });

        map.addLayer({
          id: "conservationAuthority",
          type: "line",
          source: "conservationAuthority",
          layout: {},
          paint: {
            "line-color": "grey", // Border color
            "line-width": 2, // Border width
            "line-dasharray": [2, 2],
          },
        });
        map.addLayer({
          id: "conservationAuthority-border-highlight",
          type: "line",
          source: "conservationAuthority",
          layout: {},
          paint: {
            "line-color": "#DC4633", // Border color
            "line-width": 3.5, // Border width
          },
          filter: ["==", ["get", "LEGAL_NAME"], ""],
        });

        map.addLayer({
          id: "uppertier-border-highlight",
          type: "line",
          source: "uppertier",
          layout: {},
          paint: {
            "line-color": "#DC4633", // Border color
            "line-width": 4, // Border width
          },
          filter: ["==", ["get", "CDNAME"], ""],
        });
        map.addLayer({
          id: "municipalities-border-highlight",
          type: "line",
          source: "municipalities",
          layout: {},
          paint: {
            "line-color": "#DC4633", // Border color
            "line-width": 4, // Border width
          },
          filter: ["==", ["get", "CSDNAME"], ""],
        });
      });
      // Create pop-up
      const popup = new maplibregl.Popup({
        closeButton: true,
        closeOnClick: true,
        maxWidth: "none",
      });
      map.on("mouseenter", "municipalities", () => {
        map.getCanvas().style.cursor = "pointer";
      });
      map.on("mouseleave", "municipalities", () => {
        map.getCanvas().style.cursor = "";
      });
      map.on("click", "municipalities", (e) => {
        if (cityList.includes(e.features[0].properties.CSDNAME.toUpperCase())) {
          jurisdictionInfo = "City of " + e.features[0].properties.CSDNAME;
        } else {
          jurisdictionInfo = "Town of " + e.features[0].properties.CSDNAME;
        }
        municipalFilter = [];
        // get the filtered results

        selectedJurisdiction = governmentList(jurisdictionInfo)[0];
        regionalFilter = governmentList(jurisdictionInfo)[1];
        conservationFilter = governmentList(jurisdictionInfo)[2];

        selectedCon = "";
        selectedReg = "";

        let clean = jurisdictionInfo
          .replace("City of ", "")
          .replace("Town of ", "")
          .replace("Regional Municipality of ", "");

        map.setFilter("uppertier-border-highlight", ["==", ["get", "CDNAME"], clean]);
        map.setFilter("municipalities-border-highlight", [
          "==",
          ["get", "CSDNAME"],
          clean,
        ]);
        map.setFilter("conservationAuthority-border-highlight", [
          "==",
          ["get", "LEGAL_NAME"],
          jurisdictionInfo,
        ]);

        //$: title = e.features[0].properties.CSDNAME;

        popupContent = true;
      });
    }
  });

  // Geocoder for people to input their address and zoom to input address
  const baseUrl =
    "https://nominatim.openstreetmap.org/search.php?format=jsonv2&q=";

  let query = ""; //This is the input address from users.
  let lat;
  let lon;
  let results;

  const getResults = async () => {
    results = await fetch(baseUrl + query).then((res) => res.json());
    if (results.length > 0) {
      //this is to remove the previous address point searched (if true)
      if (map.getSource(`address ${lon}`)) {
        map.removeSource(`address ${lon}`);
        map.removeLayer(`address-layer ${lon}`);
      }

      //get long - lat
      lat = +results[0].lat;
      lon = +results[0].lon;
      console.log(lat, lon);
      map.flyTo({
        // These options control the ending camera position: centered at
        // the target, at zoom level 16, and north up.
        center: [lon, lat],
        zoom: 16,
        bearing: 0,
        // These options control the flight curve, making it move slowly and zoom out almost completely before starting
        // to pan.
        speed: 2, // make the flying slow
        curve: 1, // change the speed at which it zooms out
        // This can be any easing function: it takes a number between 0 and 1 and returns another number between 0 and 1.
        easing(t) {
          return t;
        },
        // this animation is considered essential with respect to prefers-reduced-motion
        essential: true,
      });
      // add point to show the searched address
      map.addSource(`address ${lon}`, {
        type: "geojson",
        data: {
          type: "Point",
          coordinates: [lon, lat],
        },
      });
      map.addLayer({
        id: `address-layer ${lon}`,
        type: "circle",
        source: `address ${lon}`,
        paint: {
          "circle-radius": 8,
          "circle-color": "#FF0000", // Set the color of the point
        },
      });
      selectedJurisdiction = [];
      regionalFilter = [];
      conservationFilter = [];
      municipalFilter = [];
      var userPoint = turf.point([lon, lat]);
      for (let i = 0; i < municipalities.features.length; i++) {
        //console.log(turf.booleanPointInPolygon(userPoint, municipalities.features[i]))
        //console.log(municipalities.features[i])
        if (turf.booleanPointInPolygon(userPoint, municipalities.features[i])) {
          selectedJurisdiction = [];
          regionalFilter = [];
          conservationFilter = [];
          if (
            cityList.includes(
              municipalities.features[i].properties.CSDNAME.toUpperCase(),
            )
          ) {
            jurisdictionInfo =
              "City of " + municipalities.features[i].properties.CSDNAME;
          } else {
            jurisdictionInfo =
              "Town of " + municipalities.features[i].properties.CSDNAME;
          }
          selectedJurisdiction = governmentList(jurisdictionInfo)[0];
          regionalFilter = governmentList(jurisdictionInfo)[1];
          conservationFilter = governmentList(jurisdictionInfo)[2];
        }
        selectedReg = "";
        selectedMun = "";
        selectedCon = "";
      }
    } else {
      alert("Sorry, no geocoding results for " + query);
    }
  };


</script>

<main>
  
  <div id="map" />
  <div class="intro">
    <h1>GTA Flood Data Equity</h1>
    <p1> # of Municipal Flood Data Layers </p1> <br>
    <span class="dot" style= "background-color: grey"><p style="font-weight: bold; color: white; margin-left: 9px;"><b>0</p></span>
    <span class="dot" style= "background-color: #a9d6e5"><p style="font-weight: bold; color: black; margin-left: 5px;">1+</p></span>
    <span class="dot" style= "background-color: #89c2d9"><p style="font-weight: bold; color: black; margin-left: 4px;">3+</p></span>
    <span class="dot" style= "background-color: #2c7da0"><p style="font-weight: bold; color: white; margin-left: 4px;">4+</p></span>
    <span class="dot" style= "background-color: #2a6f97"><p style="font-weight: bold; color: white; margin-left: 4px;">7+</p></span>
    <span class="dot" style= "background-color: #013a63"><p style="font-weight: bold; color: white; margin-left: 1px;">10+</p></span>
    
    <p id="info">
      Map created by <a href="https://www.linkedin.com/in/chun-fu-liu/"
        >Michael Liu</a
      >
      and <a href="https://jamaps.github.io/about.html">Jeff Allen</a> at the
      <a href="https://schoolofcities.utoronto.ca/">School of Cities</a>
    </p>

    <p1><b> Select A Local Municipality:</b> </p1>
    <div class="bar" />

    <div id="select-wrapper">
      <select bind:value={selectedMun} on:change={selectMunDropdown}>
        {#each municipalList as mun}
          <option>{mun}</option>
        {/each}
      </select>
    </div>
    <div class="bar" />

    <p1><b> Select A Regional Municipality:</b> </p1>
    <div class="bar" />
    <div id="select-wrapper">
      <select bind:value={selectedReg} on:change={selectRegDropdown}>
        {#each regionList as reg}
          <option>{reg}</option>
        {/each}
      </select>
    </div>

    <div class="bar" />
    <p1><b> Select A Conservation Authority:</b> </p1>
    <div class="bar" />

    <div id="select-wrapper">
      <select bind:value={selectedCon} on:change={selectConDropdown}>
        {#each conservationList as con}
          <option>{con}</option>
        {/each}
      </select>
    </div>

    <div class="bar" />

    <!-- geocoder -->
    <p1><b>Search Your Address</b></p1>
    <input bind:value={query} placeholder="i.e. 100 St George St, Toronto" />
    <button on:click={getResults} disabled={query.length < 1}>Search</button>

    {#if popupContent}
      <h1>{jurisdictionInfo}</h1>
      <!-- Present Each List of CSV Links-->
      {#if selectedJurisdiction}
        {#if !jurisdictionInfo.endsWith("Authority")}
          <span id="subtitle"><b># of Layers: </b></span
          >{selectedJurisdiction.length}
        {/if}
        {#each selectedJurisdiction as entry, i}
          <p>
            <b>{i + 1}. </b><a href={entry.LINK} target="_blank"
              >{entry.LINK_TEXT}</a
            >
          </p>
        {/each}
        <p>  </p>
      {/if}
      

      <!-- Present Each List of CSV Links-->
      {#if municipalFilter}
        {#if municipalFilter.length > 0}
          <h2>Lower-Tier Municipalities ({municipalFilter.length} Layers)</h2>

          {#each municipalFilter as entry, i}
            {#if i == 0}
              <span id="subtitle"
                ><b>{municipalFilter[i].OFFICIAL_NAME}</b></span
              >
            {:else if municipalFilter[i].OFFICIAL_NAME != municipalFilter[i - 1].OFFICIAL_NAME}
              <span id="subtitle"
                ><b>{municipalFilter[i].OFFICIAL_NAME}</b></span
              >
            {/if}
            <p>
              <b>{i + 1}. </b><a href={entry.LINK} target="_blank"
                >{entry.LINK_TEXT}</a
              >
            </p>
          {/each}
          <p>  </p>
        {/if}
      {/if}
      <p>  </p>
      <!-- Regional Layers -->
      {#if regionalFilter}
        {#if regionalFilter.length > 0}
          <h2>Regional Municipalities ({regionalFilter.length} Layers)</h2>

          {#each regionalFilter as entry, i}
            {#if i == 0}
              <span id="subtitle"><b>{regionalFilter[i].OFFICIAL_NAME}</b></span
              >
            {:else if regionalFilter[i].OFFICIAL_NAME != regionalFilter[i - 1].OFFICIAL_NAME}
              <span id="subtitle"><b>{regionalFilter[i].OFFICIAL_NAME}</b></span
              >
            {/if}
            <p>
              <b>{i + 1}. </b><a href={entry.LINK} target="_blank"
                >{entry.LINK_TEXT}</a
              >
            </p>
          {/each}
          <p>  </p>
        {/if}
      {/if}
      

      {#if conservationFilter}
        {#if conservationFilter.length > 0}
          <h2>Conservation Authorities ({conservationFilter.length} Layers)</h2>

          {#each conservationFilter as entry, i}
            {#if i == 0}
              <span id="subtitle"
                ><b>{conservationFilter[i].OFFICIAL_NAME}</b></span
              >
            {:else if conservationFilter[i].OFFICIAL_NAME != conservationFilter[i - 1].OFFICIAL_NAME}
              <span id="subtitle"
                ><b>{conservationFilter[i].OFFICIAL_NAME}</b></span
              >
            {/if}
            <p>
              <b>{i + 1}. </b><a href={entry.LINK} target="_blank"
                >{entry.LINK_TEXT}</a
              >
            </p>
          {/each}
          <p>  </p>
        {/if}
      {/if}
    {/if}
    <p>   </p>
  </div>

</main>

<style>
  main {
    overflow-y: hidden;
  }

  :global(body) {
    overflow: hidden;
  }

  @font-face {
    font-family: TradeGothicBold;
    src: url("../assets/Trade Gothic LT Bold.ttf");
  }

  @font-face {
    font-family: RobotoRegular;
    src: url("../assets/Roboto-Regular.ttf");
  }

  #map {
    height: 100vh;
    width: 75vw;
    top: 0px;
    left: 25vw;
    position: absolute;
  }

  .bar {
    height: 1px;
    width: 20px;
    background-color: var(--brandDarkBlue);
    padding: 0px;
    margin: 0px;
    margin-left: 0px;
    opacity: 0.25;
  }

  .intro {
    position: absolute;
    top: 0px;
    left: 0px;
    width: 25vw;
    height: 100vh;
    font-size: 17px;
    font-family: TradeGothicBold;
    background-color: rgb(254, 251, 249, 1);
    color: #1e3765;
    padding: 10px;
    padding-right: 20px;
    border-radius: 5px;
    overflow-x: hidden;
  }

  h1 {
    font-size: 40px;
    font-family: TradeGothicBold;
    padding: 0px;
    padding-left: 4px;
    padding-top: 2px;
    padding-bottom: 3px;
    margin: 0px;
    margin-bottom: 7px;
    color: #1e3765;
    /* background-color: #F1C500; */
    /* -webkit-text-stroke: 1px #6FC7EA; */
    font-weight: bold;
  }

  h2 {
    font-size: 22px;
    font-family: TradeGothicBold;
    padding: 2px;
    margin: 0px;
    margin-top: 8px;
    /* margin-bottom: -4px; */
    color: #1e3765;
    /* -webkit-text-stroke: 1px #6FC7EA; */
    text-decoration: underline;
  }

  #subtitle {
    font-family: TradeGothicBold;
    color: #6d247a;
    font-size: 16px;
    font-weight: bold;
  }

  p {
    margin-top: 5px;
    margin-left: 30px;
    margin-bottom: 15px;
    font-family: RobotoRegular;
    font-size: 14px;
    opacity: 1;
    color: #007fa3;
    font-weight: bold;
  }

  p1 {
    padding-left: 4px;
    font-family: RobotoRegular;
    font-size: 14px;
    opacity: 1;
    color: #007fa3;
  }
  a {
    text-decoration: underline;
    color: #007fa3;
  }
  a:hover {
    color: #dc4633;
  }

  #info {
    font-size: 13px;
    padding: 0px;
    padding-left: 4px;
    margin: 0px;
    border-top: solid 1px #e7e7e7;
    margin-top: 7px;
    padding-top: 7px;
    padding-bottom: 15px;
  }

  button {
    font-size: 12px;
    width: 25vw;
    margin-bottom: 20px;
    font-family: TradeGothicBold;
    border-width: 0px;
  }

  input {
    width: 24vw;
    height: 20px;
    font-family: TradeGothicBold;
    color: #6d247a;
    border-width: 0px;
    margin-right: 5px;

    padding-left: 10px;
    padding-top: 3px;
    padding-bottom: 3px;
    font-weight: bold;
  }
  select {
    width: 25vw;
    height: 25px;
    font-family: TradeGothicBold;
    font-size: 16px;
    color: #6d247a;
    border-width: 0px;
    margin-right: 5px;
    padding-left: 10px;
    padding-top: 3px;
    padding-bottom: 3px;
    border-radius: 0;
    border: 1px;
  }
  .dot {
    height: 25px;
    width: 25px;
    padding: 5px;
    margin-left: 10px;
    margin-top: 10px;
    /*background-color: #bbb;*/
    border-radius: 50%;
    display: inline-block;
    opacity: 0.7;
  }

 
</style>
