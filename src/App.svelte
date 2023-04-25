<script>
  /**
   * I know i can move these to seperate files,
   * but motorseason is starting and i quickly wanted something to do this ;)
  */
  import GPX from 'gpx-parser-builder';
  import Leaflet from 'leaflet';
  import 'leaflet/dist/leaflet.css';

  let map;
  let marker;
  let converting = false;
  let files = [];
  let fileName = '';
  let gpx;

  function createMap(container) {
    let m = Leaflet.map(container).setView([52.2556445, 5.74598], 7);
    Leaflet.tileLayer(
      'https://{s}.basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}{r}.png',
      {
        attribution: `&copy;<a href="https://www.openstreetmap.org/copyright" target="_blank">OpenStreetMap</a>,
          &copy;<a href="https://carto.com/attributions" target="_blank">CARTO</a>`,
        subdomains: 'abcd',
        maxZoom: 14,
      }
    ).addTo(m);

    return m;
  }

  function setLocation (event) {
    if (!marker) {
      marker = Leaflet.marker(
        [event.latlng.lat, event.latlng.lng], 
        {
          'title': "Your Location",
          'draggable': true
        }
      ).addTo(map);
    } else {
      marker.setLatLng(event.latlng);
    }
   }

  function mapAction(container) {
    map = createMap(container);
    map.addEventListener('click', setLocation);
    map.addEventListener('locationfound', setLocation);

    return {
      destroy: () => {
        map.remove();
      },
    };
  }

  function download(filename, text) {
    var element = document.createElement('a');
    element.setAttribute('href', 'data:text/plain;charset=utf-8,' + encodeURIComponent(text));
    element.setAttribute('download', filename);

    element.style.display = 'none';
    document.body.appendChild(element);

    element.click();

    document.body.removeChild(element);
  }

  function calculateDistance(latitude1, longitude1, latitude2, longitude2) {
    latitude1 = convertToRadians(latitude1);
    longitude1 = convertToRadians(longitude1);
    latitude2 = convertToRadians(latitude2);
    longitude2 = convertToRadians(longitude2);
    
    let longitudeDiff = longitude2 - longitude1;
    let latitudeDiff = latitude2 - latitude1;

    // Calculate the distance in Radians.
    let distance = 2 * Math.asin(
        Math.sqrt(
          Math.pow(Math.sin(latitudeDiff / 2), 2) + Math.cos(latitude1) * Math.cos(latitude2) * Math.pow(Math.sin(longitudeDiff / 2),2)
        )
      );

    // Radius of earth in kilometers.
    let radius = 6371;

    // calculate the result
    return distance * radius;
  }

  function convertToRadians(coord) {
    return coord * 1 / (180 / Math.PI);
  }

  function sortWaypoints(waypoints)
  {
    let closestWaypoint = waypoints.toSorted((loc1, loc2) => calculateDistance(marker.getLatLng().lat, marker.getLatLng().lng, loc1.$.lat, loc1.$.lon) - calculateDistance(marker.getLatLng().lat, marker.getLatLng().lng, loc2.$.lat, loc2.$.lon))[0];
    
    return waypoints.concat(waypoints.splice(0, waypoints.indexOf(closestWaypoint)));
  }

  function convert()
  {
    converting = true;
    if (gpx.trk) {
      gpx.trk = gpx.trk.map(track => {
        track.trkseg = track.trkseg.map(tracksegment => {
          tracksegment.trkpt = sortWaypoints(tracksegment.trkpt)

          return tracksegment
        });
        
        return track;
      })
    }

    if (gpx.rte) {
      gpx.rte = gpx.rte.map(route => {
        route.rtept = sortWaypoints(route.rtept)
        
        return route;
      })
    }
    
    download(fileName, gpx.toString());
    converting = false;
  }

	$: if (files?.length) {
    let file = files[0];
    file.text().then(gpxString => {
      gpx = GPX.parse(gpxString);
    });
    fileName = file.name;
	}
</script>

<main>
  <p>Click on "Find My Location" or click on the map, after this you can drag the point to change your start position</p>
  <button type="button" on:click={() => map.locate()}>Find My Location</button>
  <p>Distance between your marker and the first point is calculated linearly, keep this in mind</p>
  <div style="height:400px;width:100%" use:mapAction />
  <p>
    <label>
      Upload a GPX file:
      <input
        accept=".gpx,text/gpx"
        bind:files
        type="file"
      />
    </label>
  </p>
  <button type="button" on:click={convert} disabled="{!fileName || !marker || converting}">Download</button>
  <p>It is is a big map, it may take a while. Please be patient</p>
</main>

<style>
</style>
