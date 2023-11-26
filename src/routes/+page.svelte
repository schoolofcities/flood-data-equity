<script>
  import { onMount, afterUpdate } from "svelte";
  import maplibregl, { LineIndexArray } from "maplibre-gl";

  import conservationAuthority from "../data/gta-conservation-authority.geo.json";
  import municipalities from "../data/gta-municipalities.geo.json";
  import uppertier from "../data/gta-upper-tier-municipalities.geo.json";
  import * as turf from '@turf/turf'; // this is for fitting the map boundary to GTA municipalities
  onMount(async () => {
    const csvUrl =
      "https://docs.google.com/spreadsheets/d/e/2PACX-1vQT7hsW3C1bVjp8xP8d-3HtXAMp8tQOUYOCxABymKbuOQP4TWkEDAB3wut7g1tO5Mw527PHFm_tn-dz/pub?gid=0&single=true&output=csv";

    try {
      const response = await fetch(csvUrl);
      var csvData = await response.text();
      csvData = csvData.replace(/\r/g, "");
      
      // Split the CSV data into an array of rows
      const rows = csvData.split("\n");

      // Parse CSV data into an array of arrays
      const parsedData = rows.map((row) => row.split(","));

      // Assuming 'id' is the common key
      const commonKeyIndex = parsedData[0].indexOf("MUNID"); // Adjust 'id' to your actual common key
      const munCountIndex = parsedData[0].indexOf("MUN_LAYER_COUNT")
      const regionCountIndex = parsedData[0].indexOf("REG_LAYER_COUNT")

      // Iterate through GeoJSON features
      municipalities.features.forEach((feature) => {
        // Find matching record in CSV data
        const matchingRecord = parsedData.find(
          (record) => record[commonKeyIndex] === feature.properties.MUNID,
        );
        
        // If a match is found, update GeoJSON properties with CSV data
        if (matchingRecord) {
          matchingRecord.forEach((value, index) => {
            // Skip the common key, assuming 'id' is not in CSV data
            if (index !== commonKeyIndex) {
              // If 
              if (index == munCountIndex || index == regionCountIndex){
                console.log(value)
                feature.properties[parsedData[0][index]] = parseInt(value);
              }
              else{
                //console.log("Other", value)
                //feature.properties[parsedData[0][index]] = value;
                //console.log(value)
              }
              
            }
          });
        }
      });

      // Now municipalities GeoJSON features have additional properties from the CSV data
    } catch (error) {
      console.error("Error fetching or processing CSV:", error);
    }
  });

  let map;
  let popupContent = false;

  function hidePopup() {
    popupContent = false;
  }
  console.log(municipalities.features[9].properties)

  let title;
  let uppertiers;
  let munLayer;
  let regLayer;
  let lowertier;
  let link;
  let linkText;
  let year;

  onMount(() => {
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
    map.addControl(scale, "bottom-left");

    // Adding zoom and rotation controls to the map
    map.addControl(new maplibregl.NavigationControl(), "top-right");

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
        id: "municipalities",
        type: "fill",
        source: "municipalities",
        layout: {},
        paint: {
          "fill-color": [
            "step",
            ["get", "MUN_LAYER_COUNT"], // Property in your GeoJSON data containing the values
            "#FF0000",
            2, // Red for values 0 or lower
            "#00FF00",
            4, // Green for values between 0 and 50
            "yellow",
            7, // yellow for 20-50
            "#0000FF",
            10, // Blue for values 50 or higher
            /* Add more stops and colors as needed */
            "#FFFFFF",
          ],
          "fill-opacity": 0.3,
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
      console.log(
        e.features[0].properties.OFFICIAL_MUNICIPAL_NAME,
        ", ",
        e.features[0].properties.MUN_LAYER_COUNT,
        ", ",
        e.features[0].properties.LINK_TEXT,
        ", ",
        e.features[0].properties.LINK,
        
      );

      $: title = e.features[0].properties.MUNICIPAL_NAME_SHORTFORM;
      $: lowertier = e.features[0].properties.OFFICIAL_MUNICIPAL_NAME;
      $: uppertiers = e.features[0].properties.UPPER_TIER_MUNICIPALITY;
      $: munLayer = e.features[0].properties.MUN_LAYER_COUNT;
      $: regLayer = e.features[0].properties.REG_LAYER_COUNT;
      $: linkText = e.features[0].properties.LINK_TEXT;
      $: link = e.features[0].properties.LINK;
      popupContent = true;
      
    });
  });



</script>

<main>
  <div id="map" />

  <div class="legend">
    <h1>Flood Data Equity</h1>
    <div class="legend-item">
      <span class="legend-color" style="background-color: #6D247A;" />
      <span class="legend-text">In place &nbsp;</span>
      <span class="legend-color" style="background-color: #DC4633;" />
      <span class="legend-text">No Longer In Place / Deaccessioned</span>
    </div>
    <p id="info">
      Map created by <a href="https://www.linkedin.com/in/chun-fu-liu/"
        >Michael Liu</a
      >
      and <a href="https://jamaps.github.io/about.html">Jeff Allen</a> at the
      <a href="https://schoolofcities.utoronto.ca/">School of Cities</a>
    </p>
  </div>

  <div class="popup">
    {#if popupContent}
      <div id="hide" on:click={hidePopup}>Click Here To Hide Content</div>
      <h2>{title}</h2>
      
      <p><span id="subtitle"><b>Region Name: </b></span>{uppertiers}</p>
      <p><span id="subtitle"><b>Upper Tier Municipality: </b></span>{uppertiers}</p>
      <p><span id="subtitle"><b># of Upper Tier Layers: </b></span>{regLayer}</p>
      <p><span id="subtitle"><b>Municipality Name: </b></span>{lowertier}</p>
      <p><span id="subtitle"><b>#Lower Layers: </b></span>{munLayer}</p>

      <p><span id="subtitle">Current Status: </span>{linkText}</p>

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
    src: url("../../assets/Trade Gothic LT Bold.ttf");
  }
  @font-face {
    font-family: RobotoRegular;
    src: url("../../assets/Roboto-Regular.ttf");
  }

  #map {
    height: 100vh;
    width: 100%;
    top: 0;
    left: 0;
    position: relative;
  }

  .popup {
    position: absolute;
    top: 145px;
    left: 10px;
    width: 290px; /* Set a fixed width for the popup */
    max-height: calc(
      100% - 200px
    ); /* Calculate the max height based on viewport height */
    /* overflow-y: scroll;  */
    background-color: rgb(254, 251, 249, 0.9);
    padding: 0px;
    padding-top: 15px;
    padding-left: 10px;
    padding-right: 20px;
    border-radius: 5px;
    box-shadow: 0 0 5px rgba(0, 0, 0, 0.3);
    overflow-x: hidden;
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
    top: 10px;
    left: 10px;
    width: 300px;
    height: 105px;
    font-size: 17px;
    font-family: TradeGothicBold;
    background-color: rgb(254, 251, 249, 0.9);
    color: #1e3765;
    padding: 10px;
    border-radius: 5px;
    box-shadow: 0 0 5px rgba(0, 0, 0, 0.3);
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
    font-size: 13px;
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
