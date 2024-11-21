<script>
    import { onMount } from 'svelte';
    import mapboxgl from 'mapbox-gl';
    import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';

    // Set the Mapbox access token
    mapboxgl.accessToken = 'pk.eyJ1IjoiY2Fzc3lycngiLCJhIjoiY20zcThzeHJtMG42ZjJtcTJuY3JhZHNrdCJ9.s8wO0D9yhoAGr3uShQ8t_g';

    onMount(async () => {
        // Initialize the map
        const map = new mapboxgl.Map({
            container: 'map',  
            style: 'mapbox://styles/mapbox/streets-v12',  
            center: [-71.0921, 42.3601],  
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

        // Add the Cambridge bike lanes source
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


    });
</script>

<h1>Bike Moving</h1>
<p>This project will be building an immersive, interactive map visualization of bike traffic in the Boston area during different times of the day.</p>

<!-- The map will be rendered inside this div -->
<div id="map" style="width: 100%; height: 500px;"></div>

<style>
    @import url('$lib/global.css');
    
    #map {
        flex: 1;
        background-color: bisque;
    }
</style>
