<script>
    import { onMount } from 'svelte';
    import mapboxgl from 'mapbox-gl';
    import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
    import * as d3 from 'd3';

    // Set the Mapbox access token
    mapboxgl.accessToken = 'pk.eyJ1IjoiY2Fzc3lycngiLCJhIjoiY20zcThzeHJtMG42ZjJtcTJuY3JhZHNrdCJ9.s8wO0D9yhoAGr3uShQ8t_g';

    // Declare necessary variables
    let stations = [];       // Holds station data
    let trips = [];          // Holds trip data
    let departures = new Map();  // Holds departure counts per station
    let arrivals = new Map();    // Holds arrival counts per station
    let map;                 // The map object
    let mapViewChanged = 0;  // Keeps track of map view changes
    let timeFilter = -1;

    // Time filter label reactive statement
    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString('en', {
        timeStyle: 'short',
    });

    // Function to convert date to minutes since midnight
    function minutesSinceMidnight(date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    // Filter trips based on selected time
    $: filteredTrips =
        timeFilter === -1
            ? trips
            : trips.filter((trip) => {
                  let startedMinutes = minutesSinceMidnight(trip.started_at);
                  let endedMinutes = minutesSinceMidnight(trip.ended_at);
                  return (
                      Math.abs(startedMinutes - timeFilter) <= 60 ||
                      Math.abs(endedMinutes - timeFilter) <= 60
                  );
              });

    // Filter arrivals based on filtered trips
    $: filteredArrivals = d3.rollup(
        filteredTrips,
        (v) => v.length,  // Count the number of trips ending at a station
        (d) => d.end_station_id
    );

    // Filter departures based on filtered trips
    $: filteredDepartures = d3.rollup(
        filteredTrips,
        (v) => v.length,  // Count the number of trips starting at a station
        (d) => d.start_station_id
    );

    // Filter stations based on filtered trips and clone station objects to avoid mutation
    $: filteredStations = stations.map((station) => {
        // Clone station to avoid modifying original station object
        station = { ...station };

        const arrivalsForStation = filteredArrivals.get(station.Number) ?? 0;
        const departuresForStation = filteredDepartures.get(station.Number) ?? 0;
        station.arrivals = arrivalsForStation;
        station.departures = departuresForStation;
        station.totalTraffic = station.arrivals + station.departures;

        return station;
    });

    // Function to get SVG circle coordinates from station data
    function getCoords(station) {
        if (!map) return { cx: 0, cy: 0 };
        const point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        const { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

    // Load station and traffic data
    async function loadData() {
        // Fetching station data
        try {
            stations = await d3.csv('https://dsc-courses.github.io/dsc106-2025-wi/labs/lab08/data/bluebikes-stations.csv');
            console.log('Bluebikes Station Data:', stations);
        } catch (error) {
            console.error('Error fetching station data:', error);
        }

        // Fetching traffic data (trips)
        try {
            trips = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv').then((trips) =>{
                for (let trip of trips) {
                    trip.started_at = new Date(trip.started_at);
                    trip.ended_at = new Date(trip.ended_at);
                }
                return trips;
            });

            // Calculate departures (trips starting at a station)
            departures = d3.rollup(
                trips,
                (v) => v.length,  // Count the number of trips starting at a station
                (d) => d.start_station_id
            );

            // Calculate arrivals (trips ending at a station)
            arrivals = d3.rollup(
                trips,
                (v) => v.length,  // Count the number of trips ending at a station
                (d) => d.end_station_id
            );

            // Adding arrivals, departures, and total traffic to the stations
            stations = stations.map((station) => {
                const id = station.Number;
                station.arrivals = arrivals.get(id) ?? 0;
                station.departures = departures.get(id) ?? 0;
                station.totalTraffic = station.arrivals + station.departures;
                return station;
            });

            console.log('Stations with traffic data:', stations);
        } catch (error) {
            console.error('Error fetching traffic data:', error);
        }
    }

    // Initialize the Map and Add Bike Network Data
    async function initMap() {
        map = new mapboxgl.Map({
            container: 'map',
            style: 'mapbox://styles/mapbox/streets-v12',
            center: [-71.0921, 42.3601],  // Centered around Boston
            zoom: 12,
            minZoom: 10,
            maxZoom: 16
        });

        await new Promise((resolve) => map.on('load', resolve));

        map.addSource('boston_route', {
            type: 'geojson',
            data: 'https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D',
        });
        map.addLayer({
            id: 'bike-network',
            type: 'line',
            source: 'boston_route',
            layout: {
                'line-join': 'round',
                'line-cap': 'round'
            },
            paint: {
                'line-color': 'rgba(0,255,0,0.4)',
                'line-width': 3,
            }
        });

        map.addSource('cambridge_route', {
            type: 'geojson',
            data: 'https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson'
        });
        map.addLayer({
            id: 'cambridge-bike-network',
            type: 'line',
            source: 'cambridge_route',
            layout: {
                'line-join': 'round',
                'line-cap': 'round'
            },
            paint: {
                'line-color': 'rgba(0,255,0, 0.4)',
                'line-width': 3
            }
        });
    }

    // Add SVG Overlay and Markers for Stations
    function addSvgOverlay() {
        const svg = d3.select('#map').append('svg')
            .attr('id', 'svg-overlay')
            .style('position', 'absolute')
            .style('z-index', 1)
            .style('width', '100%')
            .style('height', '100%')
            .style('pointer-events', 'none');

        // Function to update markers
        const updateMarkers = () => {
            svg.selectAll('circle')
                .data(filteredStations)  // Use filtered stations here
                .join('circle')
                .attr('r', d => Math.min(10 + d.totalTraffic / 1000, timeFilter === -1 ? 20 : 50))  // Adjust size based on filter
                .attr('fill', 'steelblue')
                .attr('fill-opacity', 0.6)
                .attr('stroke', 'white')
                .attr('stroke-width', 1)
                .attr('cx', d => getCoords(d).cx)
                .attr('cy', d => getCoords(d).cy);
        };

        // Update markers when filteredStations changes
        // svelte-ignore reactive_declaration_invalid_placement
                $: filteredStations, updateMarkers();
        
        // Listen for Map "Move" Event and Trigger Re-render of Markers
        map.on('move', (evt) => {
            mapViewChanged++;  // Increment to trigger re-render
            updateMarkers();
        });
        
        // Initial call to update markers
        updateMarkers();
    }

    // onMount Lifecycle: Load Data and Initialize Map
    onMount(async () => {
        await loadData();
        initMap();
        addSvgOverlay();
    });
</script>

<header style="display:flex; gap:1em; align-items:baseline">
    <h1>Bike Moving</h1>
    <label style="margin-left: auto;">
        Filter by time:
        <input type="range" id="time-slider" bind:value={timeFilter} min="-1" max="1440" step="1">
        {#if timeFilter === -1}
        <em id="anyTime"  style="display:block; font-style:italic; color:grey">(any time)</em>
        {:else}
        <em id="timeLabel">{timeFilterLabel}</em>
        {/if}
    </label>
</header>


<div id="map">
    <svg></svg>
</div>

<style>
    @import url('$lib/global.css');
    
    #map {
        flex: 1;
        position: relative;
    }

    #map svg {
        position: absolute;
        z-index: 1;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }

</style>