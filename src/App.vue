<script setup>
import { reactive, ref, onMounted } from 'vue'

let crime_url = ref('');
let dialog_err = ref(false);
let location_input = ref('');

// Data from API
let codes = ref([]);
let neighborhoods = ref([]);
let incidents = ref([]);
let filteredIncidents = ref([]);

// New incident form
let newIncident = reactive({
    case_number: '',
    date: '',
    time: '',
    code: '',
    incident: '',
    police_grid: '',
    neighborhood_number: '',
    block: ''
});
let form_error = ref('');
let form_success = ref('');
let loading = ref(false);
let loading_message = ref('');

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
            nw: {lat: 45.008206, lng: -93.217977},
            se: {lat: 44.883658, lng: -92.993787}
        },
        neighborhood_markers: [
            {location: [44.942068, -93.020521], marker: null},
            {location: [44.977413, -93.025156], marker: null},
            {location: [44.931244, -93.079578], marker: null},
            {location: [44.956192, -93.060189], marker: null},
            {location: [44.978883, -93.068163], marker: null},
            {location: [44.975766, -93.113887], marker: null},
            {location: [44.959639, -93.121271], marker: null},
            {location: [44.947700, -93.128505], marker: null},
            {location: [44.930276, -93.119911], marker: null},
            {location: [44.982752, -93.147910], marker: null},
            {location: [44.963631, -93.167548], marker: null},
            {location: [44.973971, -93.197965], marker: null},
            {location: [44.949043, -93.178261], marker: null},
            {location: [44.934848, -93.176736], marker: null},
            {location: [44.913106, -93.170779], marker: null},
            {location: [44.937705, -93.136997], marker: null},
            {location: [44.949203, -93.093739], marker: null}
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

    // Update location input when map pan/zoom ends
    map.leaflet.on('moveend', () => {
        updateLocationInput();
        filterCrimesByVisibleArea();
    });

    // Set initial location input
    updateLocationInput();
});

// FUNCTIONS
// Function called once user has entered REST API URL
async function initializeCrimes() {
    loading.value = true;
    loading_message.value = 'Loading data...';
    
    try {
        console.log('Initializing crimes from:', crime_url.value);
        
        // Fetch codes with format=plain for cleaner JSON
        console.log('Fetching codes...');
        loading_message.value = 'Fetching crime codes...';
        const codesResponse = await fetch(`${crime_url.value}/codes?format=plain`);
        
        console.log('Codes response status:', codesResponse.status, codesResponse.statusText);
        console.log('Codes response headers:', Object.fromEntries(codesResponse.headers.entries()));
        
        if (!codesResponse.ok) {
            const errorText = await codesResponse.text();
            console.error('Codes response error:', errorText);
            throw new Error(`Failed to fetch codes: ${codesResponse.status} ${codesResponse.statusText}`);
        }
        const codesText = await codesResponse.text();
        console.log('Codes response (first 200 chars):', codesText.substring(0, 200));
        try {
            codes.value = JSON.parse(codesText);
        } catch (parseError) {
            console.error('Failed to parse codes JSON:', parseError, 'Response:', codesText);
            throw new Error('Invalid JSON response from codes endpoint');
        }
        console.log('Codes loaded:', codes.value.length);
        
        // Fetch neighborhoods with format=plain for cleaner JSON
        console.log('Fetching neighborhoods...');
        loading_message.value = 'Fetching neighborhoods...';
        const neighborhoodsResponse = await fetch(`${crime_url.value}/neighborhoods?format=plain`);
        
        console.log('Neighborhoods response status:', neighborhoodsResponse.status, neighborhoodsResponse.statusText);
        if (!neighborhoodsResponse.ok) {
            const errorText = await neighborhoodsResponse.text();
            console.error('Neighborhoods response error:', errorText);
            throw new Error(`Failed to fetch neighborhoods: ${neighborhoodsResponse.status} ${neighborhoodsResponse.statusText}`);
        }
        const neighborhoodsText = await neighborhoodsResponse.text();
        console.log('Neighborhoods response (first 200 chars):', neighborhoodsText.substring(0, 200));
        try {
            neighborhoods.value = JSON.parse(neighborhoodsText);
        } catch (parseError) {
            console.error('Failed to parse neighborhoods JSON:', parseError, 'Response:', neighborhoodsText);
            throw new Error('Invalid JSON response from neighborhoods endpoint');
        }
        console.log('Neighborhoods loaded:', neighborhoods.value.length);
        
        // Fetch initial 1000 crimes
        console.log('Fetching incidents...');
        loading_message.value = 'Fetching crime incidents...';
        const incidentsResponse = await fetch(`${crime_url.value}/incidents?limit=1000`);
        
        console.log('Incidents response status:', incidentsResponse.status, incidentsResponse.statusText);
        if (!incidentsResponse.ok) {
            const errorText = await incidentsResponse.text();
            console.error('Incidents response error:', errorText);
            throw new Error(`Failed to fetch incidents: ${incidentsResponse.status} ${incidentsResponse.statusText}`);
        }
        const incidentsText = await incidentsResponse.text();
        console.log('Incidents response (first 200 chars):', incidentsText.substring(0, 200));
        try {
            incidents.value = JSON.parse(incidentsText);
        } catch (parseError) {
            console.error('Failed to parse incidents JSON:', parseError, 'Response:', incidentsText);
            throw new Error('Invalid JSON response from incidents endpoint');
        }
        console.log('Incidents loaded:', incidents.value.length);
        
        // Add neighborhood markers first
        loading_message.value = 'Adding markers to map...';
        addNeighborhoodMarkers();
        
        // Filter crimes by visible area
        filterCrimesByVisibleArea();
        
        loading.value = false;
        loading_message.value = '';
        console.log('All data loaded successfully!');
    } catch (error) {
        loading.value = false;
        loading_message.value = '';
        console.error('Error fetching data:', error);
        alert('Error connecting to API: ' + error.message + '\n\nCheck the browser console (F12) for more details.\n\nMake sure:\n1. Your API server is running on ' + crime_url.value + '\n2. CORS is enabled in your server code\n3. The URL is correct');
    }
}

// Get neighborhood name by id
function getNeighborhoodName(neighborhoodNumber) {
    const neighborhood = neighborhoods.value.find(n => n.id === neighborhoodNumber);
    return neighborhood ? neighborhood.name : `Neighborhood ${neighborhoodNumber}`;
}

// Get incident type by code
function getIncidentType(code) {
    const codeObj = codes.value.find(c => c.code === code);
    return codeObj ? codeObj.type : `Code ${code}`;
}

// Check if a neighborhood marker is within visible map bounds
function isNeighborhoodVisible(neighborhoodIndex) {
    const bounds = map.leaflet.getBounds();
    const marker = map.neighborhood_markers[neighborhoodIndex];
    if (marker) {
        const lat = marker.location[0];
        const lng = marker.location[1];
        return bounds.contains([lat, lng]);
    }
    return false;
}

// Get visible neighborhood numbers based on map bounds
function getVisibleNeighborhoods() {
    const visibleNeighborhoods = [];
    for (let i = 0; i < map.neighborhood_markers.length; i++) {
        if (isNeighborhoodVisible(i)) {
            visibleNeighborhoods.push(i + 1); // neighborhood numbers are 1-indexed
        }
    }
    return visibleNeighborhoods;
}

// Filter crimes to only show those in visible neighborhoods
function filterCrimesByVisibleArea() {
    if (incidents.value.length === 0) {
        filteredIncidents.value = [];
        return;
    }
    
    const visibleNeighborhoods = getVisibleNeighborhoods();
    filteredIncidents.value = incidents.value.filter(incident => 
        visibleNeighborhoods.includes(incident.neighborhood_number)
    );
    
    // Update markers with new crime counts
    updateMarkerPopups();
}

// Count crimes per neighborhood
function getCrimeCountForNeighborhood(neighborhoodNumber) {
    return incidents.value.filter(i => i.neighborhood_number === neighborhoodNumber).length;
}

// Add markers for each neighborhood
function addNeighborhoodMarkers() {
    if (!map.leaflet) {
        console.error('Map not initialized yet');
        return;
    }
    
    console.log('Adding neighborhood markers...');
    let markersAdded = 0;
    map.neighborhood_markers.forEach((markerData, index) => {
        const neighborhoodNumber = index + 1;
        const neighborhoodName = getNeighborhoodName(neighborhoodNumber);
        const crimeCount = getCrimeCountForNeighborhood(neighborhoodNumber);
        
        // Remove existing marker if any
        if (markerData.marker) {
            map.leaflet.removeLayer(markerData.marker);
        }
        
        // Create new marker with custom popup
        try {
            markerData.marker = L.marker(markerData.location).addTo(map.leaflet);
            markerData.marker.bindPopup(
                `<div style="text-align:center;">
                    <strong>${neighborhoodName}</strong><br>
                    <span style="font-size:1.2em;color:#1779ba;">${crimeCount}</span> crimes
                </div>`
            );
            markersAdded++;
            console.log(`Marker ${neighborhoodNumber}: ${neighborhoodName} - ${crimeCount} crimes at [${markerData.location[0]}, ${markerData.location[1]}]`);
        } catch (error) {
            console.error(`Error adding marker ${neighborhoodNumber}:`, error);
        }
    });
    console.log(`All markers added: ${markersAdded}/${map.neighborhood_markers.length}`);
}

// Update marker popups with current crime counts
function updateMarkerPopups() {
    map.neighborhood_markers.forEach((markerData, index) => {
        if (markerData.marker) {
            const neighborhoodNumber = index + 1;
            const neighborhoodName = getNeighborhoodName(neighborhoodNumber);
            const crimeCount = getCrimeCountForNeighborhood(neighborhoodNumber);
            markerData.marker.setPopupContent(`<strong>${neighborhoodName}</strong><br>Crimes: ${crimeCount}`);
        }
    });
}

// Submit new incident form
async function submitNewIncident() {
    form_error.value = '';
    form_success.value = '';
    
    // Validate all fields are filled
    if (!newIncident.case_number || !newIncident.date || !newIncident.time || 
        !newIncident.code || !newIncident.incident || !newIncident.police_grid || 
        !newIncident.neighborhood_number || !newIncident.block) {
        form_error.value = 'All fields are required';
        return;
    }
    
    try {
        const response = await fetch(`${crime_url.value}/new-incident`, {
            method: 'PUT',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                case_number: newIncident.case_number,
                date: newIncident.date,
                time: newIncident.time,
                code: parseInt(newIncident.code),
                incident: newIncident.incident,
                police_grid: parseInt(newIncident.police_grid),
                neighborhood_number: parseInt(newIncident.neighborhood_number),
                block: newIncident.block
            })
        });
        
        if (response.ok) {
            form_success.value = 'Incident added successfully!';
            // Clear form
            newIncident.case_number = '';
            newIncident.date = '';
            newIncident.time = '';
            newIncident.code = '';
            newIncident.incident = '';
            newIncident.police_grid = '';
            newIncident.neighborhood_number = '';
            newIncident.block = '';
            
            // Refresh incidents
            const incidentsResponse = await fetch(`${crime_url.value}/incidents?limit=1000`);
            incidents.value = await incidentsResponse.json();
            filterCrimesByVisibleArea();
        } else {
            const errorText = await response.text();
            form_error.value = errorText || 'Error adding incident';
        }
    } catch (error) {
        console.log('Error submitting incident:', error);
        form_error.value = 'Error connecting to server';
    }
}

// Clamp latitude to St. Paul bounds
function clampLat(lat) {
    return Math.max(map.bounds.se.lat, Math.min(map.bounds.nw.lat, lat));
}

// Clamp longitude to St. Paul bounds
function clampLng(lng) {
    return Math.max(map.bounds.nw.lng, Math.min(map.bounds.se.lng, lng));
}

// Check if input looks like lat/long coordinates
function isLatLng(input) {
    // Match patterns like "44.95, -93.10" or "44.95 -93.10" or "44.95,-93.10"
    const latLngPattern = /^-?\d+\.?\d*[,\s]+-?\d+\.?\d*$/;
    return latLngPattern.test(input.trim());
}

// Parse lat/long from input string
function parseLatLng(input) {
    const parts = input.trim().split(/[,\s]+/);
    if (parts.length >= 2) {
        const lat = parseFloat(parts[0]);
        const lng = parseFloat(parts[1]);
        if (!isNaN(lat) && !isNaN(lng)) {
            return { lat, lng };
        }
    }
    return null;
}

// Update the location input box with current map center address
function updateLocationInput() {
    const center = map.leaflet.getCenter();
    const lat = center.lat.toFixed(6);
    const lng = center.lng.toFixed(6);
    
    // Use Nominatim reverse geocoding to get address
    const nominatimUrl = `https://nominatim.openstreetmap.org/reverse?format=json&lat=${lat}&lon=${lng}`;
    
    fetch(nominatimUrl, {
        headers: {
            'User-Agent': 'StPaulCrimeApp/1.0'
        }
    })
    .then(response => response.json())
    .then(data => {
        if (data.display_name) {
            location_input.value = data.display_name;
        } else {
            // Fallback to coordinates if no address found
            location_input.value = `${lat}, ${lng}`;
        }
    })
    .catch(error => {
        console.log('Reverse geocoding error:', error);
        // Fallback to coordinates on error
        location_input.value = `${lat}, ${lng}`;
    });
}

// Handle Go button click - navigate to entered location
function goToLocation() {
    const input = location_input.value.trim();
    
    if (!input) {
        return;
    }
    
    if (isLatLng(input)) {
        // Input is lat/long coordinates
        const coords = parseLatLng(input);
        if (coords) {
            // Clamp coordinates to St. Paul bounds
            const clampedLat = clampLat(coords.lat);
            const clampedLng = clampLng(coords.lng);
            
            // Pan map to clamped location
            map.leaflet.panTo([clampedLat, clampedLng]);
        }
    } else {
        // Input is an address - use Nominatim to geocode
        const nominatimUrl = `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(input)}`;
        
        fetch(nominatimUrl, {
            headers: {
                'User-Agent': 'StPaulCrimeApp/1.0'
            }
        })
        .then(response => response.json())
        .then(data => {
            if (data && data.length > 0) {
                const lat = parseFloat(data[0].lat);
                const lng = parseFloat(data[0].lon);
                
                // Clamp coordinates to St. Paul bounds
                const clampedLat = clampLat(lat);
                const clampedLng = clampLng(lng);
                
                // Pan map to clamped location
                map.leaflet.panTo([clampedLat, clampedLng]);
            } else {
                console.log('Location not found');
            }
        })
        .catch(error => {
            console.log('Geocoding error:', error);
        });
    }
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    
    // Clean up URL - remove trailing slashes
    crime_url.value = crime_url.value.trim().replace(/\/+$/, '');
    
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        console.log('Dialog closed, initializing crimes with URL:', crime_url.value);
        initializeCrimes();
    }
    else {
        dialog_err.value = true;
        console.log('Dialog validation failed. URL:', crime_url.value, 'Valid:', url_input.checkValidity());
    }
}
</script>

<template>
    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
        <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
        <br/>
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>
    <div class="grid-container ">
        <!-- Loading Indicator -->
        <div v-if="loading" class="grid-x grid-padding-x">
            <div class="cell">
                <div class="loading-indicator">
                    <div class="spinner"></div>
                    <p><strong>{{ loading_message }}</strong></p>
                    <p class="loading-hint">Render free tier servers sleep after inactivity. The first request may take 30-60 seconds to wake up.</p>
                </div>
            </div>
        </div>
        
        <div class="grid-x grid-padding-x">
            <div id="leafletmap" class="cell auto"></div>
        </div>
        <div class="grid-x grid-padding-x location-controls">
            <div class="cell small-10 medium-11">
                <input 
                    id="location-input" 
                    type="text" 
                    v-model="location_input" 
                    placeholder="Enter address or lat, long coordinates"
                    @keyup.enter="goToLocation"
                />
            </div>
            <div class="cell small-2 medium-1">
                <button class="button expanded" type="button" @click="goToLocation">Go</button>
            </div>
        </div>
        
        <!-- New Incident Form -->
        <div class="grid-x grid-padding-x">
            <div class="cell">
                <h3 class="section-header">Add New Incident</h3>
                <div class="new-incident-form">
                    <div class="grid-x grid-padding-x">
                        <div class="cell small-6 medium-3">
                            <label>Case Number
                                <input type="text" v-model="newIncident.case_number" placeholder="e.g., 21012345" />
                            </label>
                        </div>
                        <div class="cell small-6 medium-3">
                            <label>Date
                                <input type="date" v-model="newIncident.date" />
                            </label>
                        </div>
                        <div class="cell small-6 medium-3">
                            <label>Time
                                <input type="time" v-model="newIncident.time" />
                            </label>
                        </div>
                        <div class="cell small-6 medium-3">
                            <label>Code
                                <select v-model="newIncident.code">
                                    <option value="">Select code...</option>
                                    <option v-for="code in codes" :key="code.code" :value="code.code">
                                        {{ code.code }} - {{ code.type }}
                                    </option>
                                </select>
                            </label>
                        </div>
                        <div class="cell small-12 medium-6">
                            <label>Incident Description
                                <input type="text" v-model="newIncident.incident" placeholder="Brief description" />
                            </label>
                        </div>
                        <div class="cell small-6 medium-2">
                            <label>Police Grid
                                <input type="number" v-model="newIncident.police_grid" placeholder="e.g., 87" />
                            </label>
                        </div>
                        <div class="cell small-6 medium-2">
                            <label>Neighborhood
                                <select v-model="newIncident.neighborhood_number">
                                    <option value="">Select...</option>
                                    <option v-for="n in neighborhoods" :key="n.id" :value="n.id">
                                        {{ n.id }} - {{ n.name }}
                                    </option>
                                </select>
                            </label>
                        </div>
                        <div class="cell small-12 medium-2">
                            <label>Block
                                <input type="text" v-model="newIncident.block" placeholder="e.g., 100 BLOCK MAIN ST" />
                            </label>
                        </div>
                    </div>
                    <div class="grid-x grid-padding-x">
                        <div class="cell">
                            <button class="button" type="button" @click="submitNewIncident">Submit Incident</button>
                            <p class="form-error" v-if="form_error">{{ form_error }}</p>
                            <p class="form-success" v-if="form_success">{{ form_success }}</p>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Crime Table -->
        <div class="grid-x grid-padding-x">
            <div class="cell">
                <h3 class="section-header">Crimes ({{ filteredIncidents.length }} incidents in visible area)</h3>
                <div class="table-scroll">
                    <table class="crime-table">
                        <thead>
                            <tr>
                                <th>Case Number</th>
                                <th>Date</th>
                                <th>Time</th>
                                <th>Incident Type</th>
                                <th>Neighborhood</th>
                                <th>Block</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="crime in filteredIncidents" :key="crime.case_number">
                                <td>{{ crime.case_number }}</td>
                                <td>{{ crime.date }}</td>
                                <td>{{ crime.time }}</td>
                                <td>{{ getIncidentType(crime.code) }}</td>
                                <td>{{ getNeighborhoodName(crime.neighborhood_number) }}</td>
                                <td>{{ crime.block }}</td>
                            </tr>
                            <tr v-if="filteredIncidents.length === 0">
                                <td colspan="6" class="no-data">No crimes to display. Enter API URL and pan/zoom the map.</td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
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

.location-controls {
    margin-top: 1rem;
    align-items: center;
}

#location-input {
    width: 100%;
    margin-bottom: 0;
    font-size: 0.9rem;
}

.location-controls .button {
    margin-bottom: 0;
}

.loading-indicator {
    background: #f0f8ff;
    border: 2px solid #1779ba;
    border-radius: 8px;
    padding: 2rem;
    text-align: center;
    margin: 1rem 0;
}

.loading-indicator .spinner {
    border: 4px solid #f3f3f3;
    border-top: 4px solid #1779ba;
    border-radius: 50%;
    width: 40px;
    height: 40px;
    animation: spin 1s linear infinite;
    margin: 0 auto 1rem;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

.loading-indicator p {
    margin: 0.5rem 0;
    font-size: 1rem;
    color: #333;
}

.loading-hint {
    font-size: 0.85rem !important;
    color: #666 !important;
    font-style: italic;
    margin-top: 0.5rem !important;
}

.section-header {
    font-size: 1.2rem;
    font-weight: bold;
    margin-top: 1.5rem;
    margin-bottom: 1rem;
    color: #333;
}

.new-incident-form {
    background: #f9f9f9;
    padding: 1rem;
    border-radius: 4px;
    margin-bottom: 1rem;
}

.new-incident-form label {
    font-size: 0.85rem;
    font-weight: 600;
}

.new-incident-form input,
.new-incident-form select {
    font-size: 0.9rem;
    margin-bottom: 0.5rem;
}

.form-error {
    color: #D32323;
    font-size: 0.9rem;
    margin-top: 0.5rem;
}

.form-success {
    color: #2E7D32;
    font-size: 0.9rem;
    margin-top: 0.5rem;
}

.table-scroll {
    max-height: 400px;
    overflow-y: auto;
    margin-bottom: 2rem;
}

.crime-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.85rem;
}

.crime-table thead {
    position: sticky;
    top: 0;
    background: #1779ba;
    color: white;
}

.crime-table th,
.crime-table td {
    padding: 0.5rem;
    text-align: left;
    border-bottom: 1px solid #ddd;
}

.crime-table tbody tr:hover {
    background: #f5f5f5;
}

.crime-table tbody tr:nth-child(even) {
    background: #fafafa;
}

.crime-table tbody tr:nth-child(even):hover {
    background: #f0f0f0;
}

.no-data {
    text-align: center;
    color: #666;
    font-style: italic;
    padding: 2rem !important;
}
</style>
