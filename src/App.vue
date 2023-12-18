<script setup>
import { reactive, ref, onMounted } from 'vue'

let crime_url = ref('http://localhost:8000/');
let dialog_err = ref(false);
let searchLocation = ref('');
let locationInputRef = ref(null);
let updateTimer = null;
let crimes = reactive([])
let map = reactive(
    {
        leaflet: null,
        center: {
            lat: 44.955139,
            lng: -93.102222,
            address: ''
        },
        zoom: 12,
        bounds: {
            nw: { lat: 45.008206, lng: -93.217977 },
            se: { lat: 44.883658, lng: -92.993787 }
        },
        neighborhood_markers: [
            { location: [44.942068, -93.020521], marker: null },
            { location: [44.977413, -93.025156], marker: null },
            { location: [44.931244, -93.079578], marker: null },
            { location: [44.956192, -93.060189], marker: null },
            { location: [44.978883, -93.068163], marker: null },
            { location: [44.975766, -93.113887], marker: null },
            { location: [44.959639, -93.121271], marker: null },
            { location: [44.947700, -93.128505], marker: null },
            { location: [44.930276, -93.119911], marker: null },
            { location: [44.982752, -93.147910], marker: null },
            { location: [44.963631, -93.167548], marker: null },
            { location: [44.973971, -93.197965], marker: null },
            { location: [44.949043, -93.178261], marker: null },
            { location: [44.934848, -93.176736], marker: null },
            { location: [44.913106, -93.170779], marker: null },
            { location: [44.937705, -93.136997], marker: null },
            { location: [44.949203, -93.093739], marker: null }
        ]
    }
);

// Vue callback for once <template> HTML has been added to web page
onMounted(() => {
    // Create Leaflet map (set bounds and valied zoom levels)
    map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        minZoom: 11,
        maxZoom: 18
    }).addTo(map.leaflet);
    map.leaflet.setMaxBounds([[44.883658, -93.217977], [45.008206, -92.993787]]);
    map.leaflet.on("dragend", updateInputFromMap);
    map.leaflet.on("zoomend", updateInputFromMap);

    // Get boundaries for St. Paul neighborhoods
    let district_boundary = new L.geoJson();
    district_boundary.addTo(map.leaflet);
    fetch('data/StPaulDistrictCouncil.geojson')
        .then((response) => {
            return response.json();
        })
        .then((result) => {
            result.features.forEach((value) => {
                district_boundary.addData(value);
            });
        })
        .catch((error) => {
            console.log('Error:', error);
        });
        
        map.neighborhood_markers.forEach((neighborhood) => {
        const marker = L.marker(neighborhood.location).addTo(map.leaflet);

        marker.on('click', () => {
            const neighborhoodNumber = map.neighborhood_markers.findIndex(
                (markerLocation) =>
                    markerLocation.location[0] === neighborhood.location[0] &&
                    markerLocation.location[1] === neighborhood.location[1]
            );

            const totalCrimes = calculateTotalCrimes(neighborhoodNumber + 1); // Assuming neighborhood_number starts from 1

            const content = `<p>Neighborhood: ${neighborhood_names[neighborhoodNumber + 1]}</p>` +
                            `<p>Total Crimes: ${totalCrimes}</p>`;

            L.popup()
                .setLatLng(marker.getLatLng())
                .setContent(content)
                .openOn(map.leaflet);
        });
    });
});

function calculateTotalCrimes(neighborhoodNumber) {
    // Assuming crimes data is available and has a neighborhood_number property
    return crimes.filter((crime) => crime.neighborhood_number === neighborhoodNumber).length;
}

// FUNCTIONS
// Function called once user has entered REST API URL
function initializeCrimes() {
    // TODO: get code and neighborhood data
    //       get initial 1000 crimes
    //let url = crime_url.value +"incidents?neighborhood=1";
    let url = 'http://localhost:8000/incidents?neighborhood=1';
    
    let params = {
        nieghborhood_number: 1
    }


    fetch(url)//,  {mode: 'no-cors'})//, params)
        .then((response) => {
            if (!response.ok) {
                throw new Error(`HTTP error! Status: ${response.status}`);
            }
            return response.json();
        })
        .then((data) => {
            // Assuming the response is an array of crimes
            console.log('Data received:', data);

            //crimes.splice(0, crimes.length, ...data);
            crimes.splice(0, crimes.lengh);
            data.forEach(crime => {
                crimes.push(crime);
            });

        })
        .catch((error) => {
            console.error('Error fetching data:', error.message);
            // Log any additional details available in the error object
            if (error.name === 'AbortError') {
                console.error('The fetch operation was aborted.');
            }
        });
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        initializeCrimes();
    }
    else {
        dialog_err.value = true;
    }
}

// Function to update input box text when map is panned/zoomed
async function updateInput() {
    try {
        const response = await fetch(
            `https://nominatim.openstreetmap.org/reverse?format=json&lat=${map.center.lat}&lon=${map.center.lng}`
        );

        if (!response.ok) {
            throw new Error(`HTTP error! Status: ${response.status}`);
        }

        const data = await response.json();
        const address = data.display_name;

        searchLocation.value = address;
    } catch (error) {
        console.error('Error fetching address:', error);
    }
}

function updateInputFromMap() {
    // Clear existing timer
    if (updateTimer) {
        clearTimeout(updateTimer);
    }

    // Set a new timer to update the input box text once the movement (drag or zoom) has ended
    updateTimer = setTimeout(() => {
        if (locationInputRef.value) {
            // Get the updated center after drag or zoom
            const newCenter = map.leaflet.getCenter();
            map.center.lat = newCenter.lat;
            map.center.lng = newCenter.lng;

            updateInput();
        }
    }, 500);
}

// Function to search for location and update the map
async function searchAndSetLocation() {
    try{
        const apiUrl = `https://nominatim.openstreetmap.org/search?format=json&q=${searchLocation.value}&limit=1`;
        const response = await fetch(apiUrl);
        const data = await response.json();

        if (data.length > 0) {
            const location = data[0];
            const lat = parseFloat(location.lat);
            const lng = parseFloat(location.lon);

            // Clamp input values if lat/long is outside of St. Paul's bounding box
            map.center.lat = Math.min(map.bounds.nw.lat, Math.max(map.bounds.se.lat, lat));
            map.center.lng = Math.min(map.bounds.se.lng, Math.max(map.bounds.nw.lng, lng));

            map.leaflet.setView([map.center.lat, map.center.lng], 16);
        }
    }catch (error) {
        console.error('Error fetching data:', error);
    }
}

let selectedIncidentTypes = reactive([]);
let selectedNeighborhoods = reactive([]);
let startDate = ref('');
let endDate = ref('');
let maxIncidents = ref('');


const neighborhoods = reactive([]);


// fetch('http://localhost:8000/neighborhoods', {
//   mode: 'no-cors'
// })
//   .then(response => {
//     // Access to response and headers is restricted due to 'no-cors' mode
//     // You won't be able to read response.json() or response.headers in this case
    
//     console.log('Request sent successfully, but limited access to response');
//     return response.json();
//   })
//   .catch(error => {
//     console.error('Error:', error);
//   });



function apiURL() {
    const baseUrl = 'http://localhost:8000/incidents';

    const params = {
        code: selectedIncidentTypes.map(type => incidentTypes[type]).join(','),
        neighborhood: selectedNeighborhoods.join(','),
        start_date: startDate.value,
        end_date: endDate.value,
        limit: maxIncidents.value,
    };

    // Filter out parameters with falsy values
    const queryParams = Object.entries(params)
        .filter(([key, value]) => value)
        .map(([key, value]) => `${key}=${value}`)
        .join('&');

    console.log('Constructed Parameters:', params);
    console.log('Query Parameters:', queryParams);

    return `${baseUrl}${queryParams ? `?${queryParams}` : ''}`;
}

let neighborhood_names = {
    1: 'Conway/Battlecreek/Highwood',
    2: 'Greater East Side',
    3: 'West Side',
    4: 'Daytons Bluff',
    5: 'Payne/Phalen',
    6: 'North End',
    7: 'Thomas/Dale(Frogtown)',
    8: 'Summit/University',
    9: 'West Seventh',
    10: 'Como',
    11: 'Hamline/Midway',
    12: 'St. Anthony',
    13: 'Union Park',
    14: 'Macalester-Groveland',
    15: 'Highland',
    16: 'Summit Hill',
    17: 'Capitol River'
}

let incident_type = {
    1: 'Murder',
    2: 'Rape',
    3: 'Robbery',
    4: 'Aggrevated Assault',
    5: 'Burglary',
    6: 'Theft',
    7: 'Motor Vehicle Theft',
    8: 'Assault',
    9: 'Arson',
    14: 'Criminal Damage of Property',
    18: 'Narcotics',
    26: 'Discharging in a Public Area',
    31: 'Death Investigation',
    99: 'Proactive Police Visit'
     
}



</script>

<template>
    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
        <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
        <br />
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>

    <div class="grid-container ">
        <div class="grid-x grid-padding-x">
            <div id="leafletmap" class="cell auto"></div>
        </div>
        <div class="grid-x grid-padding-x">
            <label>Enter Location:</label>
            <input v-model="searchLocation" @change="searchAndSetLocation" ref="locationInputRef" />
            <button class="button" @click="searchAndSetLocation">Go</button>
        </div>
            <table v-if="crime_url.length > 0">
            <thead>
                <tr>
                    <th>Code</th>
                    <th>Neighborhood</th>
                    <th>Incident</th>
                    <th>Date</th>
                    <th>Time</th>
                </tr>
            </thead>
            <tbody>
                <tr v-for="item in crimes">
                    <td>{{ item.code }}</td>
                    <td>{{ neighborhood_names[item.neighborhood_number] }}</td>
                    <td>{{ incident_type[Math.floor(item.code/100)] }}</td>
                    <td>{{ item.date }}</td>
                    <td>{{ item.time }}</td>
                    
                </tr>
            </tbody>
        </table>

    </div>
</template>
  
<style>
#rest-dialog {
    width: 20rem;
    margin-top: 1rem;
    z-index: 1000;
}

#leafletmap {
    height: 500px;
}

.dialog-header {
    font-size: 1.2rem;
    font-weight: bold;
}

.dialog-label {
    font-size: 1rem;
}

.dialog-input {
    font-size: 1rem;
    width: 100%;
}

.dialog-error {
    font-size: 1rem;
    color: #D32323;
}
</style>