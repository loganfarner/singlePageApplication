<script setup>
import { reactive, ref, onMounted } from 'vue'

const isNewIncidentDialogOpen = ref(false);

let crime_url = ref('');
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
            { location: [44.942068, -93.020521], marker:'null', number: '1' , name: 'Conway/Battlecreek/Highwood'}, 
            { location: [44.977413, -93.025156], marker:'null', number: '2' , name: 'Greater East Side'}, 
            { location: [44.931244, -93.079578], marker:'null', number: '3' , name: 'West Side'}, 
            { location: [44.956192, -93.060189], marker:'null', number: '4' , name: 'Daytons Bluff'}, 
            { location: [44.978883, -93.068163], marker:'null', number: '5' , name: 'Payne/Phalen'}, 
            { location: [44.975766, -93.113887], marker:'null', number: '6' , name: 'North End'},
            { location: [44.959639, -93.121271], marker:'null', number: '7' , name: 'Thomas/Dale(Frogtown)'},
            { location: [44.947700, -93.128505], marker:'null', number: '8' , name: 'Summit/University'},
            { location: [44.930276, -93.119911], marker:'null', number: '9' , name: 'West Seventh'},
            { location: [44.982752, -93.147910], marker:'null', number: '10', name: 'Como'},
            { location: [44.963631, -93.167548], marker:'null', number: '11', name: 'Hamline/Midway'},
            { location: [44.973971, -93.197965], marker:'null', number: '12', name: 'St. Anthony'},
            { location: [44.949043, -93.178261], marker:'null', number: '13', name: 'Union Park'},
            { location: [44.934848, -93.176736], marker:'null', number: '14', name: 'Macalester-Groveland'},
            { location: [44.913106, -93.170779], marker:'null', number: '15', name: 'Highland'},
            { location: [44.937705, -93.136997], marker:'null', number: '16', name: 'Summit Hill'},
            { location: [44.949203, -93.093739], marker:'null', number: '17', name: 'Capitol River'}
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
        maxZoom: 15
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
            //console.log(result);
            result.features.forEach((value) => {
                district_boundary.addData(value);
                //console.log(district_boundary.addData(value));
                //console.log(value);
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

function getNeighborhoodNumber(){
    //let location = [map.center.lat, map.center.lng];
    //console.log('location: '+ location[0] + ',' + location[1]);
    let max_lat = map.leaflet.getBounds().getNorth();
    let min_lon = map.leaflet.getBounds().getWest();
    let min_lat = map.leaflet.getBounds().getSouth();
    let max_lon = map.leaflet.getBounds().getEast();
    //console.log(map.leaflet.getBounds());
    let count = 0;
    let neighborhoodNumber = '';
    map.neighborhood_markers.forEach(neighborhood => {
        if(neighborhood.location[0] < max_lat && neighborhood.location[0] > min_lat && neighborhood.location[1] < max_lon && neighborhood.location[1] > min_lon){
            //console.log(neighborhood.name +' number:'+neighborhood.number);
            neighborhoodNumber = neighborhoodNumber+',' + neighborhood.number;
        } else {
            count++;
        }         
    })
    //console.log(neighborhoodNumber);
    if (count == 17){
        return '-10';
    }
    return neighborhoodNumber.slice(1, neighborhoodNumber.length);
}

let code_numbers = {
    0: '100,110,120',
    1: '210,220',
    2: '300,311,312,313,314,321,322,323,324,331,332,333,334,341,342,343,344,351,352,353,354,361,362,363,364,371,372,373,374',
    3: '400,410,411,412,420,421,422,430,431,432,440,441,442,450,451,452,453',
    4: '500,510,511,513,515,516,520,521,523,525,526,530,531,533,535,536,540,541,543,545,546,550,551,553,555,556,560,561,563,565,566',
    5: '600,601,603,611,612,613,614,621,622,623,630,631,632,633,640,641,642,643,651,652,653,661,662,663,671,672,673,681,682,683,691,692,693',
    6: '700,710,711,712,720,721,722,730,731,732',
    7: '810,861,862,863',
    8: '900,901,903,905,911,913,915,921,922,923,925,931,933,941,942,951,961,971,972,975,981,982',
    9: '1400,1401,1410,1415,1416,1420,1425,1426,1430,1435,1436',
    10: '1800,1810,1811,1812,1813,1814,1815,1820,1822,1823,1824,1825,1830,1835,1840,1841,1842,1843,1844,1845,1850,1855,1860,1865,1870,1880,1885',
    11: '2619',
    12: '3100',
    13: '9954,9959,9986'
}

// FUNCTIONS
// Function called once user has entered REST API URL
function initializeCrimes() {
    // TODO: get code and neighborhood data
    //       get initial 1000 crimes
    let url = crime_url+'/incidents?';
    let neighborhoodNubmers = getNeighborhoodNumber();
    let neighborhood_checkboxes = document.getElementsByClassName("checkbox");
    for (let i = 0; i<neighborhood_checkboxes.length-1; i++){
        //console.log(neighborhood_checkboxes[i].checked);
        if (neighborhood_checkboxes[i].checked == false){
            neighborhoodNubmers = neighborhoodNubmers.replace(i+1+",", "");
        }
    }
    if (neighborhood_checkboxes[16].checked == false){
            neighborhoodNubmers = neighborhoodNubmers.replace(",17", "");
        }
    if (neighborhoodNubmers.length > 0){
        url = url + 'neighborhood='+neighborhoodNubmers+'&';
    }
    let codes = '';
    let incident_checkboxes = document.getElementsByClassName("incident_checkbox");
    for (let i = 0; i<incident_checkboxes.length; i++){
        if (incident_checkboxes[i].checked == true){
            codes = codes+code_numbers[i]+',';
        }
    } 
    codes = codes.slice(0,codes.length-1);
    if (codes.length > 0){
        url = url + 'code='+codes+'&'
    }
    let limit_input = document.getElementById("limit");
    if (limit_input.value != 1000 ){
        if(limit_input.value<1){
            limit_input.value = 1;
        } else if (limit_input.value > 2000){
            limit_input.value = 2000;
        }
        url = url + 'limit='+limit_input.value+'&';
    }

    let start_date = document.getElementById("start").value;
    if (start_date.length > 0){
        url = url + 'start_date='+start_date+'&';
    }
    let end_date = document.getElementById("end").value;
    if (end_date.length > 0){
        url = url + 'end_date='+end_date;
    }

    console.log(url);

    fetch(url)
        .then((response) => {
            if (!response.ok) {
                throw new Error(`HTTP error! Status: ${response.status}`);
            }
            return response.json();
        })
        .then((data) => {
            // Assuming the response is an array of crimes
            console.log('Data received:', data.length);
            crimes.splice(0, crimes.length, ...data);
            crimes.push(data);
            return true;
        })
        .catch((error) => {
            console.error('Error fetching data:', error.message);
            // Log any additional details available in the error object
            if (error.name === 'AbortError') {
                console.error('The fetch operation was aborted.');
            }
            //document.getElementById('rest-dialog')
            return false;
        });
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (url_input.value !== '' && url_input.checkValidity()) {
        //dialog_err.value = false;
        dialog.close();
        crime_url = url_input.value;
        initializeCrimes();
    } else {
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
        initializeCrimes();
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

            map.leaflet.setView([map.center.lat, map.center.lng], 15);
        }
    }catch (error) {
        console.error('Error fetching data:', error);
    }
}

async function removeDynamicMarker(dynamicMarker){
    console.log('removing');
    map.leaflet.removeLayer(dynamicMarker);
}

async function markSpecificLocation(item){
    if (item.block.includes('AND')){
        searchLocation.value = item.block.slice(0,item.block.indexOf('AND')-1)+', Saint Paul, Minnesota';
    } else {
        searchLocation.value = item.block.replace("XX",'')+', Saint Paul, Minnesota';
    }
    console.log(searchLocation.value);

    // Icon options
    var iconOptions = {
    iconUrl: '/images/red_icon.png',
    iconSize: [25, 40]
    }

    // Creating a custom icon
    var customIcon = L.icon(iconOptions);
    
    await searchAndSetLocation();
    const dynamicMarker = L.marker([map.center.lat, map.center.lng], {icon: customIcon}).addTo(map.leaflet);
    const content = `<p>Incident: ${incident_type[Math.floor(item.code/100)]}</p>`+
                            `<p>Neighborhood: ${neighborhood_names[item.neighborhood_number]}</p>` +
                            `<p>Time: ${item.time}</p>`+
                            `<p>Date: ${item.date}</p>`+
                            `<button class='button' type='button' @click=${map.leaflet.removeLayer(dynamicMarker)}>Remove Marker</button>`;

    dynamicMarker.bindPopup(content);
    dynamicMarker.addTo(map.leaflet);
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
    26: 'Discharging Weapon in a Public Area',
    31: 'Death Investigation',
    99: 'Proactive Police Visit'
}
let rowColor = {
    1: '#ff928b',
    2: '#ff928b',
    3: '#ffac81',
    4: '#ff928b',
    5: '#ffac81',
    6: '#ffac81',
    7: '#ffac81',
    8: '#ff928b',
    9: '#ffac81',
    14: '#ffac81',
    18: '#efe9ae',
    26: '#efe9ae',
    31: '#efe9ae',
    99: '#efe9ae'
}

//++++ NEW INCIDENT POPUP LOGIC ++++//
function submitNewIncident() {
    const case_number = document.getElementById('case-number').value;
    const date = document.getElementById('date').value;
    const time = document.getElementById('time').value;
    const code = document.getElementById('code').value;
    const incident = document.getElementById('incident').value;
    const police_grid = document.getElementById('police_grid').value;
    const neighborhood_number = document.getElementById('neighborhood_number').value;
    const block = document.getElementById('block').value;

    const formattedTime = `${time}:00`;

    const newIncidentData = {
        case_number,
        date,
        time: formattedTime,
        code,
        incident,
        police_grid,
        neighborhood_number,
        block,
    };

    console.log('Sending data:', newIncidentData);

    fetch('http://localhost:8000/new-incident', {
        method: 'PUT',
        mode: 'cors',
        cache: 'no-cache',
        credentials: 'same-origin',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify(newIncidentData),
    })
    .then((response) => {
        console.log('Response status:', response.status);

        if (!response.ok) {
            throw new Error(`HTTP error! Status: ${response.status}`);
        }

        return response.json();
    })
    .then((data) => {
        console.log('Success:', data);
    })
    .catch((error) => {
        console.error('Error:', error);
    });
}

// +++ MANAGES INCIDENT POPUP +++ //
function openNewIncidentDialog() {
  const dialog = document.getElementById('new-incident-dialog');
  if (dialog) {
    dialog.showModal();
  }
}

// Function to close the new incident dialog
function closeNewIncidentDialog() {
  const dialog = document.getElementById('new-incident-dialog');
  if (dialog) {
    dialog.close();
  }
}

// ++++ DELETE LOGIC ++++ //
async function deleteCrime(crime) {
    const caseNumberToDelete = crime.case_number;

    try {
        const response = await fetch('http://localhost:8000/remove-incident', {
            method: 'DELETE',
            mode: 'cors',
            cache: 'no-cache',
            credentials: 'same-origin',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({ case_number: caseNumberToDelete }), 
        });

        if (!response.ok) {
            throw new Error(`Failed to delete crime. Status: ${response.status}`);
        }
        const result = await response.text();
        console.log('Delete success:', result);
        const crimeIndex = crimes.indexOf(crime);
        if (crimeIndex !== -1) {
            crimes.splice(crimeIndex, 1);
        }
    } catch (error) {
        console.error('Error deleting crime:', error);
    }
}


</script>

<template>
    <head>
    <meta name="viewport" content="width=device-width, initial-scale=1">

    </head>
    <body>

    <div class="top-bar" id="topnav">
        <div class="grid-container grid-padding-x align-center">
            <a @click="openNewIncidentDialog">New Incident</a>
            <a href="/">Home</a>
            <a href="/aboutTheProject.html">About</a>
        </div>
    </div>

    <div style="padding-left:16px">
    <h2 style="text-align: center;">Saint Paul Crime Tracker</h2>
    </div>

    <!-- New Incident Form -->
    <form>
        <dialog id="new-incident-dialog">
            <h2>New Incident Form</h2>
            <ol>
            <li>
                <label for="case-number">Case Number:</label>
                <input type="text" id="case-number" />
            </li>
            <li>
                <label for="date">Date:</label>
                <input type="date" id="date" />
            </li>
            <li>
                <label for="time">Time:</label>
                <input type="time" id="time" />
            </li>
            <li>
                <label for="code">Code</label>
                <input type="number" id="code"/>
            </li>
            <li>
                <label for="incident">Incident</label>
                <input type="text" id="incident"/>
            </li>
            <li>
                <label for="police_grid">Police Grid</label>
                <input type="number" id="police_grid">
            </li>
            <li>
                <label for="neighborhood_number">Neighborhood Number</label>
                <input type="number" id="neighborhood_number">
            </li>
            <li>
                <label for="block">Block</label>
                <input type="text" id="block">
            </li>
            </ol>
            <p class="dialog-error" v-if="dialog_err">Error: must enter all Fields</p>
            <button class="button" type="button" @click="submitNewIncident">Submit</button>
            <!-- <button type="button" @click="closeNewIncidentDialog">Close</button> -->
        </dialog>
    </form>

    

    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
        <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
        <br />
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>


    <div class="grid-container">
        <div class="grid-x grid-padding-x">
            <div id="leafletmap" class="cell auto"></div>
        </div>
        <div class="grid-x grid-padding-x">
            <label style="margin-top: 1rem; font-weight: bold;">Enter Location:</label>
            <input v-model="searchLocation" @change="searchAndSetLocation" ref="locationInputRef" style="height: 2rem; margin: 1rem; width: 65%;"/>
            <button class="button" @click="searchAndSetLocation" style="margin-bottom: 0rem;">Go</button>
        </div>
        <br>
        <div style="float: left;outline: auto;padding: 1rem;margin-bottom: 1rem;">
            <h6 style="text-align: center;">Select Neighborhoods to View</h6>                
                <li v-for="(item,index) in neighborhood_names" style="list-style: none;">
                    <input  type="checkbox" class="checkbox" checked>
                    <label>{{ neighborhood_names[index] }}</label>                
                </li>
        </div>
        <div style="float: left;outline: auto;padding: 1rem;">
            <h6 style="text-align: center;">Select Incident Type to View</h6>                
                <li v-for="(item,index) in incident_type" style="list-style: none;">
                    <input  type="checkbox" class="incident_checkbox" checked>
                    <label>{{ incident_type[index] }}</label>                
                </li>
        </div>
        <div style="float:left;padding: 1rem;">
            <label id="inputs">Start Date: </label>
            <input type="date" id="start" class="entryField">
            <label id="inputs">End Date: </label>
            <input type="date" id="end" class="entryField">
            <label id="inputs">Max Incidents to Show: </label>
            <p style="font-size: small;margin-bottom: 0rem;">Min: 1 & Max: 2000</p>
            <input type="number" id="limit" class="entryField" defaultValue="1000">
            <input type="submit" value="Submit" @click=initializeCrimes style="background-color: #fff;font-weight: bold;padding: 1rem;"> 
        </div>
        </div>
        <div>
        <div class="grid-container" style="clear: both;">
            <div id="rowKey">
                <p>Table Row Color Key</p>
                <p style="background-color: #ff928b;">Violent Crimes</p>
                <p style="background-color: #ffac81;">Property Crimes</p>
                <p style="background-color: #efe9ae;">Other Crimes</p>
            </div>
            <table v-if="crimes.length > 0">
                <thead>
                    <tr>
                        <th>Code</th>
                        <th>Neighborhood</th>
                        <th>Incident Type</th>
                        <th>Date</th>
                        <th>Time</th>
                        <th>Block</th>
                        <th>Delete</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="item in crimes" :style="{backgroundColor: rowColor[Math.floor(item.code/100)]}">
                        <td>{{ item.code }}</td>
                        <td>{{ neighborhood_names[item.neighborhood_number] }}</td>
                        <td>{{ incident_type[Math.floor(item.code/100)] }}</td>
                        <td style="min-width: 7rem;">{{ item.date }}</td>
                        <td>{{ item.time }}</td>
                        <td><a @click="markSpecificLocation(item)">{{ item.block }}</a></td>
                        <td><button v-if="item.code" class="button" type="button" @click="deleteCrime(item)" style="margin-bottom: 0rem; padding: 0.625rem;">Delete</button></td>
                    </tr>
                </tbody>
            </table>
        </div>
   </div>
</body>
</template>
  
<style>
#rowKey p {
    margin:0;
    text-align: center;
    font-weight: bold;
}

#rowKey {
    margin-bottom: 1rem;
    outline: auto;
    width: 11rem;
}

tbody td{
    margin: 0rem;
    padding: 0rem 0.625rem;
}

.entryField{
    width: 9rem;
}
#inputs{
    font-size: 1rem;
}

#new-incident-dialog {
  width: 800px; 
  background-color: #f1f1f1;
  max-width: 100%; 
  margin: auto; 
}

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
body {
    margin: 0;
    background-color: #f1f1f1;
    font-family: Arial, Helvetica, sans-serif;
}

#topnav {
    overflow: hidden;
    background-color: royalblue;

}

#topnav a {
    color: #f2f2f2;
    text-align: center;
    padding: 1rem;
    text-decoration: none;
    font-size: 24px;
    margin: 0rem 6rem;
}

#topnav a:hover {
    background-color: #ddd;
    color: black;
}

#topnav a.active {
    color: white;
}

</style>