
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Wattwise – Smart Solar Estimation</title>

  <!-- Leaflet CSS -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css">
  <link rel="stylesheet" href="https://unpkg.com/leaflet-draw/dist/leaflet.draw.css">

  <style>
    .search-container {
  position: absolute;
  top: 80px;
  left: 10px;
  z-index: 1000;
  background: white;
  padding: 8px;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.3);
  width: 250px;
}

.search-input {
  width: 100%;
  padding: 8px 12px;
  border: 2px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  box-sizing: border-box;
}

.search-input:focus {
  outline: none;
  border-color: #1e8e3e;
}
    body {
      font-family: Arial;
      margin: 0;
      background: #f4f6f8;
    }

    header {
      background: #1e8e3e;
      color: white;
      padding: 15px;
      text-align: center;
      font-size: 22px;
    }

    #map {
      height: 450px;
    }

    .container {
      padding: 15px;
      max-width: 900px;
      margin: auto;
    }

    .card {
      background: white;
      padding: 15px;
      margin-top: 15px;
      border-radius: 8px;
      box-shadow: 0 2px 6px rgba(0,0,0,0.1);
    }

    .value {
      color: #1e8e3e;
      font-weight: bold;
    }

    footer {
      text-align: center;
      font-size: 12px;
      padding: 10px;
      color: #666;
    }

    /* Zoom display styling */
    .zoom-display {
      position: absolute;
      top: 60px; /* below layer toggle */
      left: 10px;
      background: white;
      padding: 4px 8px;
      border-radius: 4px;
      box-shadow: 0 1px 4px rgba(0,0,0,0.3);
      font-weight: bold;
      z-index: 1000;
    }
  </style>
</head>
<body>

<header>
  ⚡ Wattwise
  <div style="font-size:14px;">Smart Solar Estimation Platform</div>
</header>

<div id="map"></div>
<div class="search-container">
  <input type="text" class="search-input" id="searchInput" placeholder="🔍 Search location (city, address)...">
</div>

<div class="container">
  <div class="card" id="result"> 
    <h3>Solar Estimation Result</h3>
    <p>How to use:</p>
    <p>1. Click on a polygon or rectangle tool to select your area.</p>
    <p>2. Place points on the map to outline your area.</p>
    <p>3. Complete the polygon by connecting the last point to the first.</p>
    <p>4. View estimated solar system size, installation cost, subsidy, and potential savings.</p>
  </div>
</div>

<footer>
  © 2026 Wattwise<br>
  All calculations are approximate. Final installation cost, subsidy,
  and generation depend on site survey, DISCOM rules, and equipment selection.
</footer>

<!-- WhatsApp floating button -->
<a href="https://wa.me/919405815228" target="_blank"
   style="
     position: fixed;
     bottom: 20px;
     right: 20px;
     background: #25D366;
     color: white;
     padding: 12px 16px;
     border-radius: 50px;
     text-decoration: none;
     font-weight: bold;
     box-shadow: 0 2px 6px rgba(0,0,0,0.3);
   ">
  💬 WhatsApp Us
</a>

<!-- Leaflet JS -->
<script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
<script src="https://unpkg.com/leaflet-draw/dist/leaflet.draw.js"></script>
<script src="https://unpkg.com/leaflet-geometryutil"></script>

<script>
  // Base layers
  var streetMap = L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; OpenStreetMap contributors',
    maxZoom: 21
  });

  var satelliteMap = L.tileLayer('https://{s}.google.com/vt/lyrs=y,h&x={x}&y={y}&z={z}', {
    maxZoom: 21,
    subdomains:['mt0','mt1','mt2','mt3'],
    attribution: '&copy; Google'
  });

  // Initialize map
  var map = L.map('map', {
    center: [20.5937, 78.9629],
    zoom: 5,
    maxZoom: 21,
    layers: [streetMap]
  });

  // Layer control
  var baseMaps = {
    "Street Map": streetMap,
    "Satellite": satelliteMap
  };
  var layerControl = L.control.layers(baseMaps).addTo(map);

  // Add zoom percentage below the layer toggle
  var zoomDisplay = L.DomUtil.create('div', 'zoom-display');
  layerControl.getContainer().appendChild(zoomDisplay);

  // Function to convert Leaflet zoom (1-21) → 10%-100%
  function getZoomPercentage() {
    var minZoom = 1, maxZoom = 21;
    var zoom = map.getZoom();
    var percent = Math.round(((zoom - minZoom) / (maxZoom - minZoom)) * 90 + 10); // scale to 10%-100%
    return percent;
  }

  // Initialize zoom display
  zoomDisplay.innerHTML = 'Zoom: ' + getZoomPercentage() + '%';

  map.on('zoomend', function() {
    zoomDisplay.innerHTML = 'Zoom: ' + getZoomPercentage() + '%';
  });

  // Feature group for drawn polygons
  var drawnItems = new L.FeatureGroup();
  map.addLayer(drawnItems);

  var drawControl = new L.Control.Draw({
    draw: {
      polygon: true,
      rectangle: true,
      circle: false,
      marker: false,
      polyline: false
    },
    edit: { featureGroup: drawnItems }
  });
  map.addControl(drawControl);

  // Polygon/Rectangle drawing event
  map.on(L.Draw.Event.CREATED, function (e) {
    drawnItems.clearLayers();
    drawnItems.addLayer(e.layer);

    var latlngs = e.layer.getLatLngs()[0] || e.layer.getLatLngs();
    var area = L.GeometryUtil.geodesicArea(latlngs);
    var areaSqM = area.toFixed(1);

    var solarKW = areaSqM / 5;
    var costPerKW = solarKW <= 3 ? 65000 : solarKW <= 6 ? 58000 : 52000;
    var totalCost = solarKW * costPerKW;
    var subsidy = solarKW <= 3 ? solarKW * 14588 : 78000;
    var finalCost = totalCost - subsidy;

    var monthlyUnits = solarKW * 4 * 30;
    var yearlyUnits = monthlyUnits * 12;
    var unitPrice = 6;
    var monthlySavings = monthlyUnits * unitPrice;
    var yearlySavings = yearlyUnits * unitPrice;

    document.getElementById("result").innerHTML = `
      <h3>Solar Estimation Result</h3>
      <p><b>Selected Area:</b> <span class="value">${areaSqM} m²</span></p>
      <p><b>Recommended System Size:</b> <span class="value">${solarKW.toFixed(1)} kW</span></p>
      <hr>
      <p><b>Approx Installation Cost:</b> ₹${totalCost.toFixed(0)}</p>
      <p><b>Govt Subsidy:</b> ₹${subsidy.toFixed(0)}</p>
      <p><b>Estimated Payable Cost:</b> <span class="value">₹${finalCost.toFixed(0)}</span></p>
      <hr>
      <p><b>Monthly Generation:</b> ${monthlyUnits.toFixed(0)} units</p>
      <p><b>Yearly Generation:</b> ${yearlyUnits.toFixed(0)} units</p>
      <p><b>Monthly Savings:</b> ₹${monthlySavings.toFixed(0)}</p>
      <p><b>Yearly Savings:</b> <span class="value">₹${yearlySavings.toFixed(0)}</span></p>
    `;
  });
  // Search functionality
var searchInput = document.getElementById('searchInput');
var geocodingAPI = 'https://nominatim.openstreetmap.org/search';

searchInput.addEventListener('keypress', function(e) {
  if (e.key === 'Enter') {
    var query = this.value;
    if (query.length > 2) {
      fetch(geocodingAPI + '?q=' + encodeURIComponent(query) + '&format=json&limit=1')
        .then(response => response.json())
        .then(data => {
          if (data.length > 0) {
            var lat = parseFloat(data[0].lat);
            var lon = parseFloat(data[0].lon);
            map.setView([lat, lon], 16);
            searchInput.value = data[0].display_name.split(',')[0];
          } else {
            alert('Location not found');
          }
        })
        .catch(() => alert('Search failed'));
    }
  }
});
</script>

</body>
</html>
