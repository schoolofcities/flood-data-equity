<script>
  import { onMount, afterUpdate } from "svelte";
  import maplibregl, { LineIndexArray } from "maplibre-gl";
  import { csvParse } from "d3-dsv";
  import conservationAuthority from "../data/gta-conservation-authority.geo.json";
  import municipalities from "../data/gta-municipalities.geo.json";
  import uppertier from "../data/gta-upper-tier-municipalities.geo.json";
  import * as turf from "@turf/turf"; // this is for fitting the map boundary to GTA municipalities
  import lookupTable from "../data/lookupTable.json";
  import Select from "svelte-select";
  import Papa from "papaparse";

  let isContentVisible = true;
  const csvLink =
    "https://docs.google.com/spreadsheets/d/e/2PACX-1vQT7hsW3C1bVjp8xP8d-3HtXAMp8tQOUYOCxABymKbuOQP4TWkEDAB3wut7g1tO5Mw527PHFm_tn-dz/pub?gid=0&single=true&output=csv";
  const region =
    "https://docs.google.com/spreadsheets/d/e/2PACX-1vQT7hsW3C1bVjp8xP8d-3HtXAMp8tQOUYOCxABymKbuOQP4TWkEDAB3wut7g1tO5Mw527PHFm_tn-dz/pub?gid=193533627&single=true&output=csv";
  const consAuth =
    "https://docs.google.com/spreadsheets/d/e/2PACX-1vQT7hsW3C1bVjp8xP8d-3HtXAMp8tQOUYOCxABymKbuOQP4TWkEDAB3wut7g1tO5Mw527PHFm_tn-dz/pub?gid=898330427&single=true&output=csv";
  // loads a list of unique jurisdictions (i.e. municipality, region, conservation authority)
  let popupContent = false;
  function hidePopup() {
    popupContent = false;
  }
  let jurisdictionInfo;
  let conservation_csvData;
  let map;
  let title; // the title of the popup
  let uppertiers; // region name
  let munLayer; // municipality name
  let regLayer; // # of layers in the region
  let lowertier; // # of layers in the municipality
  let conservation;
  let conservationFilter;
  let municipalFilter;
  let regionalFilter;
  let selectedJuristidction = "";
  let municipalData = [] // storing municipal data and links
  let regionalData = [] // storing regional data and links
  let conservationData = [] // storing conservation authority data and links
  let municipal; 
  let c = ['a','b', 'c']
  // ============================functions=============================================

  onMount(async ()=> {
    municipalData = await processCsv(csvLink)
    regionalData = await processCsv(region)
    conservationData = await processCsv(consAuth)

    municipal = municipalData.data
    regionalData = regionalData.data
    conservationData = conservationData.data
  })


  //console.log(municipalData)
  //console.log(regionalData)
  //console.log(conservationData)
  // this is for handling the imported data and joining with the GIS layer
  async function handleCsvData(data, header) {
    // Assuming 'MUNID' is the common key and find the index of regional and municipal layer count
    const commonKeyIndex = header.indexOf("MUNID");
    const munCountIndex = header.indexOf("MUN_LAYER");
    const regionCountIndex = header.indexOf("REG_WIDE_LAYER");

    try {
      municipalities.features.forEach((feature) => {
        // Find matching record in CSV data
        const matchingRecord = data.find(
          (obj) => obj.MUNID.toString() === feature.properties.MUNID,
        );
        // If a match is found, update GeoJSON properties with CSV data
        if (matchingRecord) {
          feature.properties[header[munCountIndex]] = parseInt(
            matchingRecord.MUN_LAYER,
          );
          feature.properties[header[regionCountIndex]] = parseInt(
            matchingRecord.REG_WIDE_LAYER,
          );
        }
      });
      return true;
      // Now municipalities GeoJSON features have additional properties from the CSV data
    } catch (error) {
      console.error("Error fetching or processing CSV:", error);
      return false;
    }
  }

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

  // loading lookuptable to create unique jurisdiction lists.
  function jurisDictionList(jurisdiction) {
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

  // this is for filtering a list of relevant jurisdiction that users select in their dropdown list
  function govFiltering(lookupTable, filtergov, government, jurisdiction) {
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
    // loop through each jurisdiction that is within the jurisdiction list, for each jurisdiction,
    // add the fitting rows into the matching list
    let matchingList = [];
    for (let j = 0; j < jurisdictionList.length; j++) {
      for (let i = 0; i < csvData.length; i++) {
        if (csvData[i].OFFICIAL_NAME === jurisdictionList[j]) {
          matchingList.push(csvData[i]);
        }
      }
    }
    return matchingList;
  }

  async function governmentList(jurisdiction, csv) {
    let region = [];
    let conservation = [];
    let municipal = [];

    if (jurisdiction.startsWith("Regional")) {
      municipal = govFiltering(
        lookupTable,
        "Regional_Municipality",
        "Municipality",
        jurisdiction,
      );
      conservation = govFiltering(
        lookupTable,
        "Regional_Municipality",
        "Conservation_Authority",
        jurisdiction,
      );

      //load data from each csv, this avoids having to load all csvs when running.
      let municipalData = await processCsv(csvLink);
      let conservationData = await processCsv(consAuth);

      municipalFilter = dataFiltering(municipalData.data, municipal);
      conservationFilter = dataFiltering(conservationData.data, conservation);

      return municipalFilter
    } 
    else if (jurisdiction.endsWith("Authority")) {
      region = govFiltering(
        lookupTable,
        "Conservation_Authority",
        "Regional_Municipality",
        jurisdiction,
      );
      municipal = govFiltering(
        lookupTable,
        "Conservation_Authority",
        "Municipality",
        jurisdiction,
      );

      let municipalData = await processCsv(csvLink);
      let regionData = await processCsv(region);

      municipalFilter = dataFiltering(municipalData.data, municipal);
      regionalFilter = dataFiltering(regionData.data, region);

      return regionalFilter
    } else {
      region = govFiltering(
        lookupTable,
        "Municipality",
        "Regional_Municipality",
        jurisdiction,
      );
      conservation = govFiltering(
        lookupTable,
        "Municipality",
        "Conservation_Authority",
        jurisdiction,
      );

      let conservationData = await processCsv(consAuth);
      let regionData = await processCsv(region);

      regionalFilter = dataFiltering(regionData.data, region);
      conservationFilter = dataFiltering(conservationData.data, conservation);

      return conservationFilter
    }
  }

  // this is a function for filtering data when clicked on the region/municipality
  /*
  function filtering(inputcsv, fieldName, matchingRecord, filtered) {
    function filterMunicipality(inputcsv) {
      if (fieldName == "UPPER_TIER_MUNICIPALITY") {
        return (
          inputcsv[fieldName] === matchingRecord[fieldName] &&
          inputcsv["OFFICIAL_MUNICIPAL_NAME"] == ""
        );
      } else {
        return (
          inputcsv[fieldName] === matchingRecord[fieldName] &&
          inputcsv[fieldName] != ""
        );
      }
    }

    filtered = inputcsv.filter(filterMunicipality);
    return filtered;
  }
*/
  function selectDropdown(e) {
    let list = []
    list = governmentList(e.detail.value);
    console.log(e.detail.value);
    popupContent = true;
    jurisdictionInfo = e.detail.value
    
    


  
}

  // ============================Loading Data and Maps=============================================

  let conservationList = jurisDictionList("Conservation_Authority");
  let regionList = jurisDictionList("Regional_Municipality");
  let municipalList = jurisDictionList("Municipality");

  onMount(async () => {
    // only load the maps when the google sheet data is loaded
    // read the csvfile from google sheets

    const csv = await processCsv(csvLink);
    const dataLoaded = handleCsvData(csv.data, csv.meta.fields);

    if (dataLoaded) {
      map = new maplibregl.Map({
        container: "map",
        style: "https://basemaps.cartocdn.com/gl/positron-gl-style/style.json", //'https://basemaps.cartocdn.com/gl/positron-gl-style/style.json',
        center: [-79.4, 44.1], // starting position
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

      // Adding zoom and rotation controls to the map
      map.addControl(new maplibregl.NavigationControl(), "top-right");
      map.addControl(scale, "bottom-left");
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
              "#a9d6e5",
              2, // Red for values 0 or lower
              "#89c2d9",
              3,
              "#2c7da0",
              4, // Green for values between 0 and 50
              "#2a6f97",
              7, // yellow for 20-50
              "#013a63",
              10, // Blue for values 50 or higher
              /* Add more stops and colors as needed */
              "#FFFFFF",
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
            "line-width": 5, // Border width
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
        municipalFilter = filtering(
          result.data,
          "OFFICIAL_MUNICIPAL_NAME",
          e.features[0].properties,
          municipalFilter,
        );
        regionalFilter = filtering(
          result.data,
          "UPPER_TIER_MUNICIPALITY",
          e.features[0].properties,
          regionalFilter,
        );

        $: title = e.features[0].properties.MUNICIPAL_NAME_SHORTFORM;
        $: lowertier = e.features[0].properties.MUNICIPAL_NAME_SHORTFORM;
        $: uppertiers = e.features[0].properties.UPPER_TIER_MUNICIPALITY;
        $: munLayer = e.features[0].properties.MUN_LAYER;
        $: regLayer = e.features[0].properties.REG_WIDE_LAYER;

        popupContent = true;
      });
      map.on("click", "conservationAuthority-fill", (e) => {
        console.log(e.features[0].properties.LEGAL_NAME);
        $: conservation = e.features[0].properties.LEGAL_NAME;

        conservationFilter = filtering(
          conservation_csvData,
          "CA_Name",
          e.features[0].properties,
          conservationFilter,
        );
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
        //console.log(map.getLayer("address"))
        map.removeSource(`address ${lon}`);
        map.removeLayer(`address-layer ${lon}`);
        //console.log("Removed Source")
      }
      //get long - lat
      lat = +results[0].lat;
      lon = +results[0].lon;
      console.log(lat, lon);
      map.flyTo({
        // These options control the ending camera position: centered at
        // the target, at zoom level 9, and north up.
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

      // Add a symbol layer
      console.log("Add Source");
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
    } else {
      alert("Sorry, no geocoding results for " + query);
    }
  };
</script>

<main>
  <div id="map" />
  <div class="legend">
    <h1>Flood Data Equity</h1>

    <p id="info">
      Map created by <a href="https://www.linkedin.com/in/chun-fu-liu/"
        >Michael Liu</a
      >
      and <a href="https://jamaps.github.io/about.html">Jeff Allen</a> at the
      <a href="https://schoolofcities.utoronto.ca/">School of Cities</a>
    </p>
    <p>
      <i>Select A Local Municipality:</i>
    </p>
    <div class="bar" />

    <div id="select-wrapper">
      <Select
        id="select"
        items={municipalList}
        value={selectedJuristidction}
        clearable={false}
        showChevron={true}
        on:input={selectDropdown}
        --background="white"
        --selected-item-color="#6D247A"
        --width="25vw"
        --height="22px"
        --item-color="#6D247A"
        --border-radius="0"
        --border="1px"
        --list-border-radius="0px"
        --font-size="14.45px"
        --max-height="30px"
        --item-is-active-color="#0D534D"
        --item-is-active-bg="#6FC7EA"
      />
    </div>
    <div class="bar" />
    <p>
      <i>Select A Regional Municipality:</i>
    </p>
    <div class="bar" />

    <div id="select-wrapper">
      <Select
        id="select"
        items={regionList}
        value={selectedJuristidction}
        clearable={false}
        showChevron={true}
        on:input={selectDropdown}
        --background="white"
        --selected-item-color="#6D247A"
        --width="25vw"
        --height="22px"
        --item-color="#6D247A"
        --border-radius="0"
        --border="1px"
        --list-border-radius="0px"
        --font-size="14.45px"
        --max-height="30px"
        --item-is-active-color="#0D534D"
        --item-is-active-bg="#6FC7EA"
      />
    </div>
    <div class="bar" />
    <p>
      <i>Select A Conservation Authority:</i>
    </p>
    <div class="bar" />

    <div id="select-wrapper">
      <Select
        id="select"
        items={conservationList}
        value={selectedJuristidction}
        clearable={false}
        showChevron={true}
        on:input={selectDropdown}
        --background="white"
        --selected-item-color="#6D247A"
        --width="25vw"
        --height="25px"
        --item-color="#6D247A"
        --border-radius="0"
        --border="1px"
        --list-border-radius="0px"
        --font-size="14.45px"
        --max-height="30px"
        --item-is-active-color="#0D534D"
        --item-is-active-bg="#6FC7EA"
      />
    </div>
    <div class="bar" />
    <p></p>
    <p></p>

    <input bind:value={query} placeholder="Search for a location" />
    <button on:click={getResults} disabled={query.length < 1}>Search</button>

    {#if popupContent}
      {console.log(municipal)}
      {jurisdictionInfo}
        {#each municipal as entry (entry.ID)}
          {#if entry.OFFICIAL_NAME == jurisdictionInfo}
              <p><b>Name: </b>{entry.OFFICIAL_NAME}</p>
              <p><b>Data: </b>{entry.LINK_TEXT}</p>
          {/if}
        {/each}


      <h2>{title}</h2>
      <p><span id="subtitle"><b>Region: </b></span>{uppertiers}</p>
      <p>
        <span id="subtitle"><b># of Region-Wide Layers: </b></span>{regLayer}
      </p>
      <p><span id="subtitle"><b>Municipality: </b></span>{lowertier}</p>
      <p><span id="subtitle"><b># of Municipal Layers: </b></span>{munLayer}</p>
      <p>
        <span id="subtitle"><b>Conservation Authority: </b></span>{conservation}
      </p>
      <p>
        <span id="subtitle"><b># of Conservation Authority Layers: </b></span
        >{munLayer}
      </p>
    {/if}
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
    width: 100%;
    top: 0px;
    left: 0px;
    position: absolute;
  }

  .popup {
    position: absolute;
    top: 53vh;
    left: 0px;
    width: 25vw; /* Set a fixed width for the popup */
    max-height: calc(
      100% - 200px
    ); /* Calculate the max height based on viewport height */
    /* overflow-y: scroll;  */
    background-color: rgb(254, 251, 249, 0.9);
    padding: 0px;
    padding-top: 15px;
    padding-left: 10px;
    padding-right: 20px;
    padding-bottom: 30px;
    border-radius: 5px;
    box-shadow: 0 0 5px rgba(0, 0, 0, 0.3);
    overflow-x: hidden;
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
  #hide {
    position: absolute;
    height: 15px;
    width: 100%;
    padding: 0px;
    margin: 0px;
    top: 0px;
    left: 0px;
    font-family: TradeGothicBold;
    font-size: 12px;
    color: #9da9bd;
    background-color: none;
    border-bottom: solid 1px rgb(227, 227, 227);
    z-index: 9999;
    text-align: center;
  }
  #hide:hover {
    color: #dc4633;
    cursor: pointer;
  }

  .legend {
    position: absolute;
    top: 0px;
    left: 0px;
    width: 25vw;
    height: 100vh;
    font-size: 17px;
    font-family: TradeGothicBold;
    background-color: rgb(254, 251, 249, 0.9);
    color: #1e3765;
    padding: 10px;
    padding-right: 20px;
    border-radius: 5px;
    box-shadow: 0 0 5px rgba(0, 0, 0, 0.3);
    overflow-x: hidden;
  }

  h1 {
    font-size: 24px;
    font-family: TradeGothicBold;
    padding: 0px;
    padding-left: 4px;
    padding-top: 2px;
    border-bottom: solid 1px #e7e7e7;
    padding-bottom: 3px;
    margin: 0px;
    margin-bottom: 7px;
    color: #00a189;
    /* background-color: #F1C500; */
    background-color: #ffffff;
    /* -webkit-text-stroke: 1px #6FC7EA; */
  }

  h2 {
    font-size: 24px;
    font-family: TradeGothicBold;
    padding: 2px;
    margin: 0px;
    margin-top: 8px;
    /* margin-bottom: -4px; */
    color: #00a189;
    /* -webkit-text-stroke: 1px #6FC7EA; */
  }

  #subtitle {
    font-family: TradeGothicBold;
    color: #1e3765;
    font-size: 16px;
  }

  p {
    font-family: RobotoRegular;
    font-size: 14px;
    opacity: 1;
    color: #1e3765;
  }

  a {
    text-decoration: underline;
    color: #1e3765;
  }
  a:hover {
    color: #dc4633;
  }

  #info {
    font-size: 11.2px;
    padding: 0px;
    margin: 0px;
    border-top: solid 1px #e7e7e7;
    margin-top: 7px;
    padding-top: 7px;
  }

  .legend-item {
    display: flex;
    align-items: center;
    margin-bottom: 5px;
  }

  .legend-color {
    width: 13px;
    height: 13px;
    margin-right: 5px;
    border-radius: 50%;
  }

  .legend-text {
    font-family: TradeGothicBold;
    color: #1e3765;
    font-size: 14px;
  }

  .map-overlay-dropdown {
    position: absolute;
    font: 17px/20px "Trade Gothic LT Bold";
    background: rgba(249, 249, 249, 1);
    height: 45px;
    bottom: 195px;
    width: 157px;
    margin: 10px 0 0 10px;
    padding: 10px;
    border-radius: 5px;
    box-shadow: 0 0 5px rgba(0, 0, 0, 0.3);
    overflow: visible;
  }
</style>
