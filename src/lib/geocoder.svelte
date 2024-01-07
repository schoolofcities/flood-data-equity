<script>
	const baseUrl = 'https://nominatim.openstreetmap.org/search.php?format=jsonv2&q='
	let query = '';
	let lat;
	let lon;

	let results;
	const getResults = async () => {
		results = await fetch(baseUrl + query).then((res) => 
			res.json()
		);
        if(results.length > 0) {
            lat = +results[0].lat
            lon = +results[0].lon
        }
        else {alert("Sorry, no geocoding results for " + query)}
	};

</script>
<h2>Geocoding with Nominatim</h2>

<input bind:value={query} placeholder="Search for a location" />
<button on:click={getResults} disabled={query.length < 1}>Search</button>

<p>Latitude: {lat ? lat : ''}</p>
<p>Longitude: {lon ? lon : ''}</p>

