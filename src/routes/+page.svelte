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

    // Step 1: Function to get SVG circle coordinates from station data
    function getCoords(station) {
        if (!map) return { cx: 0, cy: 0 };
        const point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        const { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

    // Step 2: Load Station and Traffic Data
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
            trips = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv');
            
            // Calculate departures (trips starting at a station)
            departures = d3.rollup(
                trips,
                (v) => v.length,  // Count the number of trips starting at a station
                (d) => d.start_station_id  // Key by start_station_id
            );

            // Calculate arrivals (trips ending at a station)
            arrivals = d3.rollup(
                trips,
                (v) => v.length,  // Count the number of trips ending at a station
                (d) => d.end_station_id  // Key by end_station_id
            );

            // Adding arrivals, departures, and total traffic to the stations
            stations = stations.map((station) => {
                let id = station.Number; // Assuming station ID is stored in `Number`
                
                // Add the number of arrivals, if any, otherwise set to 0
                station.arrivals = arrivals.get(id) ?? 0;
                
                // Add the number of departures, if any, otherwise set to 0
                station.departures = departures.get(id) ?? 0;
                
                // Total traffic is the sum of arrivals and departures
                station.totalTraffic = station.arrivals + station.departures;
                
                return station;
            });

            console.log('Stations with traffic data:', stations);
        } catch (error) {
            console.error('Error fetching traffic data:', error);
        }
    }

    // Step 3: Initialize the Map and Add Bike Network Data
    async function initMap() {
        // Initialize the map
        map = new mapboxgl.Map({
            container: 'map',
            style: 'mapbox://styles/mapbox/streets-v12',
            center: [-71.0921, 42.3601],  // Centered around Boston
            zoom: 12,
            minZoom: 10,
            maxZoom: 16
        });

        await new Promise((resolve) => map.on('load', resolve));

        // Adding bike network (Boston route)
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

        // Adding Cambridge bike network (Cambridge route)
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

    // Step 4: Add SVG Overlay and Markers for Stations
    function addSvgOverlay() {
        const svg = d3.select('#map').append('svg')
            .attr('id', 'svg-overlay')
            .style('position', 'absolute')
            .style('z-index', 1)
            .style('width', '100%')
            .style('height', '100%')
            .style('pointer-events', 'none');

        // Step 5: Update Markers Dynamically
        const updateMarkers = () => {
            svg.selectAll('circle')
                .data(stations)
                .join('circle')
                .attr('r', d => Math.min(10 + d.totalTraffic / 1000, 20))  // Map total traffic to circle size
                .attr('fill', 'steelblue')
                .attr('fill-opacity', 0.6)
                .attr('stroke', 'white')
                .attr('stroke-width', 1)
                .attr('cx', d => getCoords(d).cx)
                .attr('cy', d => getCoords(d).cy);
        };

        // Step 6: Listen for Map "Move" Event and Trigger Re-render of Markers
        map.on('move', (evt) => {
            mapViewChanged++; // Increment variable to trigger re-render
            updateMarkers(); // Re-render markers
        });

        // Initial call to update markers
        updateMarkers();
    }

    // onMount Lifecycle: Load Data and Initialize Map
    onMount(async () => {
        await loadData();  // Load station and traffic data
        await initMap();   // Initialize the map and add data sources
        addSvgOverlay();   // Add SVG overlay and markers
    });
</script>

<h1>Bike Moving</h1>
<p>This project builds an immersive, interactive map visualization of bike traffic in the Boston area during different times of the day.</p>

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
