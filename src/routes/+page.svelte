<script>
  import { onMount } from "svelte";
  import maplibregl from "maplibre-gl";
  import cartoBasemap from "../assets/carto-basemap.json";
  import conservationAuthority from "../data/gta-conservation-authority.geo.json";
  import municipalities from "../data/gta-municipalities.geo.json";
  import uppertier from "../data/gta-upper-tier-municipalities.geo.json";
  import * as turf from "@turf/turf"; // this is for fitting the map boundary to GTA municipalities
  import lookupTable from "../data/lookupTable.json";
  import Papa from "papaparse";
  import logo from "../assets/top-logo-full.svg";
  import logoTCO from "../assets/TCOlogo.png";
  import "../assets/maplibre-gl.css";

  const municipalCsv = "./GTA Flood Data Equity - Shared Municipal CSV.csv"
    // "https://docs.google.com/spreadsheets/d/e/2PACX-1vQT7hsW3C1bVjp8xP8d-3HtXAMp8tQOUYOCxABymKbuOQP4TWkEDAB3wut7g1tO5Mw527PHFm_tn-dz/pub?gid=0&single=true&output=csv";
  const regionCsv = "./GTA Flood Data Equity - Shared Regional CSV.csv"
    // "https://docs.google.com/spreadsheets/d/e/2PACX-1vQT7hsW3C1bVjp8xP8d-3HtXAMp8tQOUYOCxABymKbuOQP4TWkEDAB3wut7g1tO5Mw527PHFm_tn-dz/pub?gid=193533627&single=true&output=csv";
  const consAuthCsv = "./GTA Flood Data Equity - Shared Conservation Authority CSV.csv"
    // "https://docs.google.com/spreadsheets/d/e/2PACX-1vQT7hsW3C1bVjp8xP8d-3HtXAMp8tQOUYOCxABymKbuOQP4TWkEDAB3wut7g1tO5Mw527PHFm_tn-dz/pub?gid=898330427&single=true&output=csv";
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
  let dataLoaded = false;
  let query = ""; //This is the input address from users.
  let lat;
  let lon;
  let results;
  let about = true;
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

    // console.log(municipalData);
    // console.log(regionalData);
    // console.log(conservationData);
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
    // console.log(matchingList);
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
      regionalFilter = []; // have this here so that it clears out the municipalFilter values from previous selection
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
      conservationFilter = []; // have this here so that it clears out the municipalFilter values from previous selection
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
      municipalFilter = []; // have this here so that it clears out the municipalFilter values from previous selection
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
    // console.log(jurisdictionInfo);
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
    map.setFilter("uppertier-border-highlight", [
      "==",
      ["get", "CDNAME"],
      clean,
    ]);
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
    if (map.getSource(`address ${lon}`)) {
      map.removeSource(`address ${lon}`);
      map.removeLayer(`address-layer ${lon}`);
    }
  }
  // the dropdown for regional
  function selectRegDropdown() {
    selectedJurisdiction = [];
    municipalFilter = [];
    regionalFilter = [];

    // get the filtered results
    jurisdictionInfo = selectedReg;
    // console.log(jurisdictionInfo);
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
    map.setFilter("uppertier-border-highlight", [
      "==",
      ["get", "CDNAME"],
      clean,
    ]);
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
    if (map.getSource(`address ${lon}`)) {
      map.removeSource(`address ${lon}`);
      map.removeLayer(`address-layer ${lon}`);
    }
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
    map.setFilter("uppertier-border-highlight", [
      "==",
      ["get", "CDNAME"],
      clean,
    ]);
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
    if (map.getSource(`address ${lon}`)) {
      map.removeSource(`address ${lon}`);
      map.removeLayer(`address-layer ${lon}`);
    }
  }
  // ============================Loading Data and Maps=============================================

  // obtain a data for each level of government
  let conservationList = jurisDictionListing("CONSERVATION_NAME");
  let regionList = jurisDictionListing("REGION_NAME");
  let municipalList = jurisDictionListing("MUNICIPAL_NAME");

  onMount(async () => {
    // only load the maps when the google sheet data is loaded
    // read the csvfile from google sheets

    // console.log(dataLoaded);

    const csv = await processCsv(municipalCsv);
    dataLoaded = handleCsvData(csv.data, csv.meta.fields);

    if (dataLoaded) {
      map = new maplibregl.Map({
        container: "map",
        style: cartoBasemap, //'https://basemaps.cartocdn.com/gl/positron-gl-style/style.json',
        center: [-79.0, 44.1], // starting position
        zoom: 8, // starting zoom;
        minZoom: 7,
        maxZoom: 14,
        projection: "globe",
        scrollZoom: true,
        attributionControl: false,
        touchPitch: false,
        dragRotate: false, // Disable drag to pitch
        pitchWithRotate: false // Disable pitch control with rotation
      });
      map.setMaxBounds([
        [-82.0, 41.0], // Southwest coordinates
        [-76.0, 46.0]  // Northeast coordinates
      ]);

      // Adding scale bar to the map
      // Add zoom control to the top right corner
      map.addControl(new maplibregl.NavigationControl(), 'top-right');

      // Add scale bar to the bottom right corner
      map.addControl(new maplibregl.ScaleControl({
        maxWidth: 80,
        unit: 'metric' // Use 'imperial' for miles
      }), 'bottom-right');
      


      // Adding additional layers from geojson
      map.on("load", function () {
       
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

        map.addSource("municipalities", {
          type: "geojson",
          data: municipalities,
        });
        map.addSource("uppertier", {
          type: "geojson",
          data: uppertier,
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
              "lightgrey",
              1, // 1 or lower
              "#cbe0e8",
              4, // 3
              "#6FC7EA",
              8,
              "#00a6e8",
            ],
            "fill-opacity": 0.5,
          },
        });
        map.addLayer({
          id: "municipalities-border",
          type: "line",
          source: "municipalities",
          layout: {},
          paint: {
            "line-color": "#1E3765", // Border color
            "line-width": 1, // Border width
          },
        });
        map.addLayer({
          id: "uppertier-border",
          type: "line",
          source: "uppertier",
          layout: {},
          paint: {
            "line-color": "#1E3765", // Border color
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
            "line-color": "#007FA3", // Border color
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
            "line-color": "#F1C500", // Border color
            "line-width": 7, // Border width
          },
          filter: ["==", ["get", "LEGAL_NAME"], ""],
        });

        map.addLayer({
          id: "uppertier-border-highlight",
          type: "line",
          source: "uppertier",
          layout: {},
          paint: {
            "line-color": "#F1C500", // Border color
            "line-width": 7, // Border width
          },
          filter: ["==", ["get", "CDNAME"], ""],
        });
        map.addLayer({
          id: "municipalities-border-highlight",
          type: "line",
          source: "municipalities",
          layout: {},
          paint: {
            "line-color": "#F1C500", // Border color
            "line-width": 7, // Border width
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
        if (map.getSource(`address ${lon}`)) {
          map.removeSource(`address ${lon}`);
          map.removeLayer(`address-layer ${lon}`);
        }

        map.setFilter("uppertier-border-highlight", [
          "==",
          ["get", "CDNAME"],
          clean,
        ]);
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

  const getResults = async () => {
    // remove query that includes "Canada" to prevent the query from failing. 
    /*
    if (query.toLowerCase().includes("ontario")){
      query = query
    }
    else{
      query = query + ", Ontario"
    }*/
      
    results = await fetch(baseUrl + query).then((res) => res.json());
    if (results.length > 0) {
      //this is to remove the previous address point searched (if true)
      if (map.getSource(`address ${lon}`)) {
        map.removeSource(`address ${lon}`);
        map.removeLayer(`address-layer ${lon}`);
      }
      console.log(results)
      console.log(results.length)

      // limiting query to a box bound by the GTA. 
      var queryResults = []
      for (let i = 0; i <results.length; i++){
        if ((parseFloat(results[i].lon) > -80.20185 && parseFloat(results[i].lon) < -78.41080) && (parseFloat(results[i].lat) < 44.5092 && parseFloat(results[i].lat) > 43.27279))  {
          console.log(results[i])
          queryResults.push([results[i].lat, results[i].lon])
        }
      }
      // when there are multiple addresses inside the boundary. 

      lat = queryResults[0][0];
      lon = queryResults[0][1];
      console.log(lat, lon)

      //get long - lat
      
      //console.log(lat, lon);
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
      map.setFilter("uppertier-border-highlight", [
        "==",
        ["get", "CDNAME"],
        "",
      ]);
      map.setFilter("municipalities-border-highlight", [
        "==",
        ["get", "CSDNAME"],
        "",
      ]);
      map.setFilter("conservationAuthority-border-highlight", [
        "==",
        ["get", "LEGAL_NAME"],
        "",
      ]);

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
          "circle-color": "#F1C500", // Set the color of the point
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
  <!-- <div id="logo">
    <a href="https://www.schoolofcities.utoronto.ca/"
      ><img src={logo} alt="School of Cities" /></a
    > </div>-->
  

  <div class="intro">
    <h1>Taking Stock Of Data On Flooding In The GTA</h1>
   
    <p id="info">Search for data in the <span id="purple">Greater Toronto Area (GTA)</span> pertaining to flooding risk. Data availability varies substantially across the region, depending on what data municipalities and conservation areas collect and if they share their data publicly. <br><br>
      <span 
      id="about-button"
      on:click={() => {
        about = true;
    }}
      >
      Click here to read more about this page and related research
    </span>
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

    <p1> # of Flood Data Layers by Lower-Tier Municipality</p1> <br />
    <span class="dot" style="background-color: lightgrey"
      ><p style="font-weight: bold; color: white; margin-left: 9px;">
        <b>0</b>
      </p></span
    >
    <span class="dot" style="background-color: #cbe0e8"
      ><p style="font-weight: bold; color: black; margin-left: 5px;">
        1+
      </p></span
    >
    <span class="dot" style="background-color: #6fc7ea"
      ><p style="font-weight: bold; color: black; margin-left: 4px;">
        4+
      </p></span
    >
    <span class="dot" style="background-color: #00a6e8"
      ><p style="font-weight: bold; color: black; margin-left: 4px;">
        8+
      </p></span
    >
    <!-- <span class="dot" style="background-color: #2a6f97"
      ><p style="font-weight: bold; color: white; margin-left: 4px;">
        7+
      </p></span
    >
    <span class="dot" style="background-color: #013a63"
      ><p style="font-weight: bold; color: white; margin-left: 1px;">
        10+
      </p></span
    > -->

    
    

    {#if popupContent}

    <div id="available">
      <p>Available datasets for selected area:</p>
    </div>
    
      <h2 id="datatitle">{jurisdictionInfo.toUpperCase()}</h2>
      <!-- Present Each List of CSV Links-->
      {#if selectedJurisdiction}
        
          <!-- <span id="subtitlelayers"><b>{selectedJurisdiction.length} Layers</b></span
          > -->
        {#if selectedJurisdiction.length < 1}
            <p>No data available<p>
        {/if}
        
        {#each selectedJurisdiction as entry, i}
          <p>
            <b>{i + 1}. </b><a href={entry.LINK} target="_blank"
              >{entry.LINK_TEXT}</a
            >
          </p>
        {/each}
        <p></p>
      {/if}

      <!-- Present Each List of CSV Links-->
      {#if municipalFilter}
        {#if municipalFilter.length > 0}
          <h3>Lower-Tier Municipalities 
            <!-- ({municipalFilter.length} Layers) -->
          </h3>

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
          <p></p>
        {/if}
      {/if}
      <p></p>
      <!-- Regional Layers -->
      {#if regionalFilter}
        {#if regionalFilter.length > 0}
          <h3>Regional Municipalities 
            <!-- ({regionalFilter.length} Layers) -->
          </h3>

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
          <p></p>
        {/if}
      {/if}

      {#if conservationFilter}
        {#if conservationFilter.length > 0}
          <h3>Conservation Authorities
             <!-- ({conservationFilter.length} Layers) -->
            </h3>

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
          <p></p>
        {/if}
      {/if}
    {/if}
    <p></p>

    <p id="infosmall">
      <br>
      Research by Nicole Gordon, Emily Majeed, <a href="https://www.geography.utoronto.ca/people/directories/all-faculty/nidhi-subramanyam" target="_blank">Nidhi Subramanyam</a>, and <a href="https://www.environment.utoronto.ca/people/directories/all-faculty/robert-soden" target="_blank">Robert Soden</a>
      <br><br>
      Map created by <a href="https://www.linkedin.com/in/chun-fu-liu/" target="_blank"
        >Michael Liu</a
      >
      and <a href="https://jamaps.github.io/" target="_blank">Jeff Allen</a> at the
      <a href="https://schoolofcities.utoronto.ca/" target="_blank">School of Cities</a>. Basemap data from <a href="https://www.openstreetmap.org/" target="_blank">OpenStreetMap</a> hosted via <a href="https://carto.com/" target="_blank">CARTO</a>. Map built with <a href="https://maplibre.org/" target="_blank">MapLibre GL JS</a>. Page built with <a href="https://svelte.dev/" target="_blank">Svelte</a>.
      <br>
    </p>
    <div id="logo">
      <a href="https://www.schoolofcities.utoronto.ca/" target="_blank"
        ><img src={logo} alt="School of Cities" /></a
      >
      &nbsp; &nbsp; 
      <a href="https://www.climateobservatory.ca/" target="_blank"
        ><img src={logoTCO} alt="Toronto Climate Observatory" /></a
      >
      <br><br><br>
    </div>
    

</main>


{#if about}
<div class="container">
<div class="floating">
  <button
  id="application-button"
  on:click={() => {
      about = false;

  }}
  >Click to close and view the map
      </button>
  <h1>Taking stock of data on flooding in the GTA</h1>
  <p>
    On December 15, 2023, The Toronto Star published a <a href="https://www.thestar.com/interactives/torontos-secret-flood-map-your-home-could-be-at-risk-of-flooding-and-youd-never/article_1e35fe6a-8a1e-11ee-a228-ab1e8de2cf96.html" target="_blank">report on Toronto’s secret flood map</a>, which revealed that most Torontonians do not have access to publicly available open data to assess if their homes are at risk of flooding. The unavailability of such data or the limited usability of existing data is a factor preventing people from taking flood risk mitigation measures such as moving away from floodplains, retrofitting their homes to reduce flooding, insuring their property and belongings, or demanding action from the government to prevent damaging impacts during future flooding events. 
  </p>
  <p>
    To help assess the availability and quality of flooding-related data and information in the city or region, our team ‘scavenged’ multiple government websites and open data portals. We found various flooding datasets available in the public domain at the municipal level in the Greater Toronto Area (GTA). We also identified plans to mitigate flood risk. The map shows these plans and datasets by municipality, <a href="https://conservationontario.ca/conservation-authorities/about-conservation-authorities" target="_blank">conservation area</a>, and by street address. Overall, this map shows that the openly available flood data are largely historical; they do not predict how flooding will vary in the future with climate change. Few, if any, adequately characterize the differential impacts of flooding on vulnerabilized communities. Many of these datasets also fail to mention how the underlying data were generated and if there was any community input in the process. Similarly, not all data are maintained and updated regularly. Very few municipalities (e.g., Durham Region) provide guidance about what residents can do before, during, or after a flood to reduce impacts. 
  </p>
  <p>
    To ensure that the GTA is resilient in the face of future flooding events, municipalities and higher tiers of government should open up additional information on changing precipitation patterns, existing flooding hotspots, damages from past flooding events, communities vulnerable to flooding, as well as infrastructures being developed to mitigate flooding risk. This information ecosystem should be updated regularly to ensure that flooding projections account for changing land use and development patterns as well as infrastructural interventions aimed at reducing flooding. Flooding is costly. Thousands of individuals pay the price for avoidable flood impacts and damages. The unavailability or privatization of data on projected flooding impacts shouldn’t add to the costs of building a flood-resilient GTA.
  </p>
  <p>
    This project is part of a broader effort by the <a href="https://www.climateobservatory.ca/" target="_blank"
    >Toronto Climate Observatory</a> and the <a href="https://www.schoolofcities.utoronto.ca/" target="_blank"
    >School of Cities</a> to assess the inequitable distribution of flooding across the Greater Toronto Area. Research has demonstrated that vulnerabilized communities, such as Indigenous, racialized, or economically disadvantaged, are most impacted by climate hazards, and struggle the most to recover in their aftermath. Yet too little has been done in our region to fully assess flood risk from this perspective, a necessary first step to guide develop an effective response. By evaluating existing information on flooding, this project aims to identify priority information needs that our work, and that of others, may contribute to meeting.
  </p>
  <p>
    We will periodically update this flooding data repository and welcome your comments, critiques, and suggestions.
  </p>
  <h2>
    Research Credits
  </h2>
  <p>
    Nicole Gordon, Master of Planning student, Toronto Metropolitan University
  </p>
  <p>
    Emily Majeed, 2024 M.Sc. in Planning graduate, University of Toronto
  </p>
  <p>
    <a href="https://www.geography.utoronto.ca/people/directories/all-faculty/nidhi-subramanyam" target="_blank">Nidhi Subramanyam</a>, Assistant Professor, Department of Geography & Planning, University of Toronto
  </p>
  <p>
    <a href="https://www.environment.utoronto.ca/people/directories/all-faculty/robert-soden" target="_blank">Robert Soden</a>, Assistant Professor, Department of Computer Science, and School of the Environment, University of Toronto
  </p>
  <br>
  <p>
  <a href="https://www.schoolofcities.utoronto.ca/" target="_blank"
        ><img src={logo} alt="School of Cities" /></a
      >
      &nbsp; &nbsp; 
      <a href="https://www.climateobservatory.ca/" target="_blank"
        ><img src={logoTCO} alt="Toronto Climate Observatory" /></a
      >
    </p>
  <br>
  <br>
  <button
  id="application-button"
  on:click={() => {
      about = false;

  }}
  >Click to close and view the map
      </button>
</div>
</div>

{/if}



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

  .intro {
    position: absolute;
    top: 0px;
    left: 0px;
    width: 25vw;
    min-width: 325px;
    height: 100vh;
    font-size: 17px;
    font-family: TradeGothicBold;
    background-color: #f8fbfc;
    /* background-color: rgb(254, 251, 249, 1); */
    border-right: solid 1px #6FC7EA;
    color: #1e3765;
    padding: 10px;
    padding-right: 20px;
    overflow-x: hidden;
  }

  @media screen and (max-width: 600px) {
    #map {
      width: 100vw;
      height: 50vh;
      top: 0;
      left: 0;
      position: absolute;
    }

    .intro {
      width: calc(100vw - 30px);
      height: 50vh;
      top: 50vh;
      left: 0;
      min-width: 0;
      border-top: solid 1px #6FC7EA;
      border-right: none;
      position: absolute;
    }
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

  h1 {
    font-size: 34px;
    font-family: TradeGothicBold;
    padding: 0px;
    padding-left: 4px;

    /*padding-bottom: 3px;*/
    margin: 0px;
    margin-bottom: 0px;
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
    color: #007fa3;
    /* -webkit-text-stroke: 1px #6FC7EA; */
    text-decoration: underline;
  }

  .intro h1 {
    /* color: #007fa3; */
    color: #1e3765;
    background-color: #eaf9ff;
    text-decoration: none;
    font-size: 23px;
    padding: 5px;
    padding-left: 15px;
    padding-right: 15px;
    margin-top: 5px;
    margin-bottom: 12px;
    font-style: italic;
    border: solid 1px #6FC7EA;
    border-bottom-right-radius: 25px;
    border-top-left-radius: 25px;
  }

  h3 {
    font-size: 20px;
    font-family: TradeGothicBold;
    padding-left: 4px;
    margin: 0px;
    margin-top: 8px;
    margin-bottom: 12px;
    text-decoration: underline;
    /* margin-bottom: -4px; */
    color: #1e3765;
    /* -webkit-text-stroke: 1px #6FC7EA; */
  }
  #subtitle {
    font-family: TradeGothicBold;
    color: #6d247a;
    font-size: 18px;
    padding-left: 4px;
    font-weight: bold;
  }

  #subtitlelayers {
    font-family: TradeGothicBold;
    color: #1e3765;
    font-size: 16px;
    font-weight: bold;
  }

  #datatitle {
    font-family: TradeGothicBold;
    text-decoration: none;
    color: #6d247a;
    font-size: 18px;
    font-weight: bold;
    margin-top: 15px;
    padding-top: 10px;
  }

  #purple {
    color: #6d247a;
  }

  p {
    margin-top: 5px;
    margin-left: 5px;
    margin-bottom: 15px;
    font-family: RobotoRegular;
    font-size: 14px;
    opacity: 1;
    color: #007fa3;
    /* font-weight: bold; */
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
    font-size: 15px;
    padding: 0px;
    padding-left: 4px;
    margin: 0px;
    border-top: solid 1px #e7e7e7;
    margin-top: 7px;
    padding-top: 7px;
    padding-bottom: 15px;
  }

  #infosmall {
    font-size: 12px;
    padding: 0px;
    padding-left: 4px;
    margin: 0px;
    border-top: solid 1px #e7e7e7;
    margin-top: 7px;
    padding-top: 7px;
    padding-bottom: 15px;
  }

  #available {
    margin-top: 20px;
    margin-bottom: -10px;
    border-top: solid 1px #6FC7EA;
  }

  button {
    font-size: 12px;
    width: 25vw;
    min-width: 330px;
    margin-bottom: 20px;
    font-family: RobotoRegular;
    border-width: 0px;
  }

  input {
    width: 25vw;
    min-width: 317px;
    height: 20px;
    font-family: RobotoRegular;
    color: #6d247a;
    border-width: 0px;
    margin-right: 0px;

    padding-left: 10px;
    padding-top: 3px;
    padding-bottom: 3px;
    font-weight: bold;
  }
  select {
    width: 25vw;
    min-width: 330px;
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

  #logo {
    position: absolute;
    width: 550px;
    height: 5vh;
    /* right: 0px; */
    /* bottom: 0px; */
    /* background-color: rgb(254, 251, 249, 0.5); */
    z-index: 6;
    opacity: 0.8;
  }
  #logo:hover {
    opacity: 1;
  }
  img {
    height: 5vh;
    color: blue;
  }
  .container {
    /* border: 0px solid #dddddd; */
    width: 100vw;
    height: 100vh;
    left: 0px;
    top: 0px;
    position: absolute;
    background-color: #ffffff;
    opacity: 0.97;
    z-index: 9999999999999999;
  }

  .floating {
    height: 90vh;
    max-width: 650px;
    margin: 0 auto;
    margin-top: 5vh;
    align-items: center;
    justify-content: center;
    text-align: center;
    background-color: white;
    border: solid 1px #6FC7EA;
    opacity: 1;
    overflow-y: scroll;
    scrollbar-width: 1px;
    z-index: inherit;
  }
  .floating h1 {
    margin: 0 auto;
    max-width: 700px;
    margin-left: 20px;
    margin-right: 20px;
    margin-top: 20px;
    margin-bottom: 7px;
    padding-left: 25px;
    padding-top: 5px;
    padding-bottom: 5px;
    padding-right: 5px;
    border: solid 1px #6FC7EA;
    border-bottom-right-radius: 30px;
    border-top-left-radius: 30px;
    position: relative;
    text-align: left;
    color: #1e3765;
    background-color: #eaf9ff;
    /* text-decoration: underline; */
    font-size: 26px;
    font-style: italic;
  }
  .floating h2 {
    margin: 0 auto;
    max-width: 700px;
    margin-left: 0px;
    margin-right: 10px;
    padding-left: 20px;
    padding-top: 30px;
    position: relative;
    text-decoration: none;
    text-align: left;
    color: #1e3765;
    font-size: 22px;
    font-style: italic;
  }

  .floating p {
    margin: 0 auto;
    margin-left: 10px;
    margin-right: 10px;
    /* max-width: 700px; */
    padding-bottom: 5px;
    padding-top: 10px;
    padding-left: 10px;
    padding-right: 10px;
    position: relative;
    font-size: 17px;
    text-align: left;
    color: #1e3765;
    font-family: RobotoRegular, sans-serif;
    font-size: 16px;
    text-decoration: none;
    line-height: 1.5;
  }
  /* SCROLL BARS */
  ::-webkit-scrollbar {
    width: 5px;
  } /* Track */
  ::-webkit-scrollbar-track {
    box-shadow: inset 0 0 5px grey;
    border-radius: 5px;
  }

  /* Handle */
  ::-webkit-scrollbar-thumb {
    background: #41729f;
    border-radius: 5px;
  }

  /* Handle on hover */
  ::-webkit-scrollbar-thumb:hover {
    background: #41729f;
  }
  #application-button{
    width: 100%;
    height: 20px;
    font-size: 15px;
    font-weight: bold;
    margin-bottom: 0px;
    background-color: #41729f;
    color: white;
    opacity: 0.4
  }
  #application-button:hover {
    cursor: pointer;
    opacity: 1;

  }
  #about-button{
    text-decoration: underline;
    color: #1e3765;
  }
  #about-button:hover {
    cursor: pointer;
    color: #dc4633;

  }
  
</style>
