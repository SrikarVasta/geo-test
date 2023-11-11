<script>
  import { onMount } from "svelte";
  import L from "leaflet";
  import * as PIXI from "pixi.js";
  import "leaflet-pixi-overlay";
  import "leaflet-geometryutil";

  let map;
  let mapContainer;
  let startTime;
  let project;
  let renderer;

  const createMap = (container) => {
    const map = L.map(container).setView([51.5074, -0.1278], 8);
    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution: "© OpenStreetMap contributors",
    }).addTo(map);
    return map;
  };

  function destination(latlng, heading, distance) {
    heading = (heading + 360) % 360;
    var rad = Math.PI / 180,
      radInv = 180 / Math.PI,
      R = L.CRS.Earth.R, // approximation of Earth's radius
      lon1 = latlng.lng * rad,
      lat1 = latlng.lat * rad,
      rheading = heading * rad,
      sinLat1 = Math.sin(lat1),
      cosLat1 = Math.cos(lat1),
      cosDistR = Math.cos(distance / R),
      sinDistR = Math.sin(distance / R),
      lat2 = Math.asin(
        sinLat1 * cosDistR + cosLat1 * sinDistR * Math.cos(rheading)
      ),
      lon2 =
        lon1 +
        Math.atan2(
          Math.sin(rheading) * sinDistR * cosLat1,
          cosDistR - sinLat1 * Math.sin(lat2)
        );
    lon2 = lon2 * radInv;
    lon2 = lon2 > 180 ? lon2 - 360 : lon2 < -180 ? lon2 + 360 : lon2;
    console.log(latlng, rad);
    return L.latLng([lat2 * radInv, lon2]);
  }

  function generateSquarePath(startPoint, numPoints, speedRange) {
    // Generate random speed within the specified range
    var speed = Math.random() * (speedRange[1] - speedRange[0]) + speedRange[0];

    // Calculate destination point on the opposite side of the Earth

    var destinationPoint = destination(
      {
        lat: startPoint[0],
        lng: startPoint[1],
      },
      180,
      40075
    );
    // Generate intermediate points
    var path = [];
    for (var i = 0; i <= numPoints; i++) {
      var fraction = i / numPoints;
      var lat =
        startPoint[0] + fraction * (destinationPoint.lat - startPoint[0]);
      var lon =
        startPoint[1] + fraction * (destinationPoint.lng - startPoint[1]);
      path.push([lat, lon]);
    }

    return { path: path, speed: speed };
  }

  function generateCircularPath(center, radius, numPoints, speedRange) {
    // Generate random speed within the specified range
    var speed = Math.random() * (speedRange[1] - speedRange[0]) + speedRange[0];

    // Generate circular path
    var path = [];
    for (var i = 0; i < numPoints; i++) {
      var angle = (i / numPoints) * 2 * Math.PI;
      var lat = center[0] + radius * Math.sin(angle);
      var lon = center[1] + radius * Math.cos(angle);
      path.push([lat, lon]);
    }

    return { path: path, speed: speed };
  }

  const makeSquare = () => {
    const markerGraphics = new PIXI.Graphics();
    markerGraphics.beginFill(0xff0011);
    markerGraphics.drawRect(0, 0, 80, 80); // (x, y, width, height)
    markerGraphics.endFill();
    return markerGraphics;
  };

  onMount(() => {
    map = createMap(mapContainer); // Adjust zoom level to view both cities

    const markerGraphics = makeSquare();
    const manchesterLatLng = [53.483959, -2.244644];
    const londonLatLng = [51.5074, -0.1278];

    const pixiContainer = new PIXI.Container();
    pixiContainer.addChild(markerGraphics);

    const pixiOverlay = L.pixiOverlay((utils) => {
      const container = utils.getContainer();
      renderer = utils.getRenderer();
      project = utils.latLngToLayerPoint;
      const markerCoords = project(manchesterLatLng);
      markerGraphics.x = markerCoords.x;
      markerGraphics.y = markerCoords.y;

      renderer.render(container);
    }, pixiContainer);

    pixiOverlay.addTo(map);

    // Animation
    startTime = performance.now();
    let returning = false;
    var geojson = {
      type: "FeatureCollection",
      features: [
        {
          type: "Feature",
          properties: {},
          geometry: {
            coordinates: [
              [53.17838730321125, -2.3900698459790988],
              [53.13265656132526, -2.3366857231972915],
              [53.01352855156239, -2.3443120264516324],
              [52.93086216177528, -2.1765333548529213],
              [52.825002672488296, -2.146028141835359],
              [52.71888464521027, -2.077391412544756],
              [52.60787685889824, -2.05451250278162],
              [52.52443590867378, -1.9477442572191421],
              [52.39897668488675, -1.7952181921291697],
              [52.32446169996717, -1.7952181921291697],
              [52.268492969987676, -1.6579447335479358],
              [52.18908282255464, -1.46728715218606],
              [52.067357142987305, -1.3376399968592239],
              [51.992283682956554, -1.2156191447879792],
              [51.87943706598867, -1.162235022006115],
              [51.7568661176235, -1.2079928415335814],
              [51.52970459260999, -1.2766295708229904],
              [51.40618327098318, -1.3071347838416898],
              [51.26324292527315, -1.3147610870960875],
              [51.27278614915406, -1.0478404731892397],
              [51.310939229040684, -0.7961724657899083],
              [51.37763085135853, -0.6207674909368279],
              [51.41569678472058, -0.46061512259245774],
              [51.46323466411033, -0.30046275424811597],
            ],
            type: "LineString",
          },
        },
      ],
    };

    // Parse geoJSON data to get path
    var squarePathData = generateSquarePath(manchesterLatLng, 100, [50, 80]);
    const circularPath = generateCircularPath(
      manchesterLatLng,
      1500,
      100,
      [110, 300]
    );

    console.log({ circularPath });

    var path = circularPath.path;
    // Function to calculate new position
    function calculateNewPosition(progress) {
      // Ensure progress is within [0, 1]
      progress = Math.max(0, Math.min(1, progress));

      var index = Math.floor(progress * (path.length - 1));

      // Ensure index is within [0, path.length - 2]
      index = Math.max(0, Math.min(path.length - 2, index));
      var start = path[index];
      var end = path[index + 1];

      var segmentProgress = progress * (path.length - 1) - index;
      var lat = start[0] + (end[0] - start[0]) * segmentProgress;
      var lng = start[1] + (end[1] - start[1]) * segmentProgress;

      return { lat: lat, lng: lng };
    }

    function animate(time) {
      var progress = (time - startTime) / 10000;
      var newPos = calculateNewPosition(progress);

      //animate
      // markerGraphics.clear();
      markerGraphics.beginFill(0xff0000);
      markerGraphics.drawRect(newPos.lat, newPos.lng, 80, 80);
      markerGraphics.endFill();
      var markerCoords = project([newPos.lat, newPos.lng]);
      markerGraphics.x = markerCoords.x;
      markerGraphics.y = markerCoords.y;
      if (progress < 1) {
        // Continue the animation as long as the destination is not reached
        requestAnimationFrame(animate);
      } else {
        // If the destination is reached, reverse the path and start again
        path.reverse();
        startTime = performance.now();
        requestAnimationFrame(animate);
      }
      renderer.render(pixiContainer);
    }

    requestAnimationFrame(animate);
  });
</script>

<div id="map" bind:this={mapContainer} />

<style>
  #map {
    height: 100vw;
  }
</style>
