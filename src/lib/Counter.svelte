<script>
  import { onMount } from "svelte";
  import L from "leaflet";
  import * as PIXI from "pixi.js";
  import "leaflet-pixi-overlay";
  import "leaflet-geometryutil";
  import * as turf from "@turf/turf";

  let map;
  let mapContainer;
  let startTimeSquare, startTimeCircle;
  let project;
  let renderer;

  const createMap = (container) => {
    const map = L.map(container).setView(
      [56.94783079082404, 26.214645326303554],
      7
    );
    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution: "© OpenStreetMap contributors",
    }).addTo(map);
    return map;
  };

  function destination(latlng, heading, distance) {
    heading = (heading + 360) % 360;
    let rad = Math.PI / 180,
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
    return L.latLng([lat2 * radInv, lon2]);
  }

  function generateSquarePath(startPoint, numPoints, speedRange) {
    let speed = Math.random() * (speedRange[1] - speedRange[0]) + speedRange[0];

    let destinationPoint = destination(
      {
        lat: startPoint[0],
        lng: startPoint[1],
      },
      90, //
      1000075
    );
    // Generate intermediate points
    let path = [];
    for (let i = 0; i <= numPoints; i++) {
      let fraction = i / numPoints;
      let lat =
        startPoint[0] + fraction * (destinationPoint.lat - startPoint[0]);
      let lon =
        startPoint[1] + fraction * (destinationPoint.lng - startPoint[1]);
      path.push([lat, lon]);
    }
    const point1 = L.latLng(path[0]);
    const point2 = L.latLng(path[path.length - 1]);
    const distance = point1.distanceTo(point2);
    const duration = distance / speed; 

    return { path: path, duration: duration, speed: speed };
  }

  function genCirclePath(start, radius, speedRange) {
    const speed =
      Math.random() * (speedRange[1] - speedRange[0]) + speedRange[0];

    function generateCircularPath(center, numPoints) {
      const newRadius = Math.random() * (radius[1] - radius[0]) + radius[0];
      const options = { steps: numPoints, units: "kilometers" };
      const circle = turf.circle(center, newRadius, options);
      return circle.geometry.coordinates;
    }

    function calculateLineLength(line) {
      let length = 0;
      for (let i = 1; i < line.length; i++) {
        const point1 = line[i - 1];
        const point2 = line[i];
        const segmentLength = turf.distance(
          turf.point(point1),
          turf.point(point2),
          { units: "kilometers" }
        );
        length += segmentLength;
      }
      return length;
    }
    const path = generateCircularPath(start, 1000);
    console.log(path, "path");
    const distance = calculateLineLength(path);
    const duration = distance / speed;

    return { path: path[0], duration: duration, speed: speed };
  }

  const makeSquare = () => {
    const markerGraphics = new PIXI.Graphics();
    markerGraphics.beginFill(0x4545ec);
    markerGraphics.drawRect(0, 0, SCALE, SCALE); // (x, y, width, height)
    markerGraphics.endFill();
    return markerGraphics;
  };
  const SCALE = 15;
  const redrawSquare = (markerGraphics) => {
    markerGraphics.beginFill(0x4545ec);
    markerGraphics.drawRect(0, 0, SCALE, SCALE); // (x, y, width, height)
    markerGraphics.endFill();
    return markerGraphics;
  };

  const redrawCircle = (markerGraphics) => {
    markerGraphics.beginFill(0xff0000);
    markerGraphics.drawCircle(0, 0, SCALE / 2); // (x, y, width, height)
    markerGraphics.endFill();
    return markerGraphics;
  };

  onMount(() => {
    map = createMap(mapContainer); // Adjust zoom level to view both cities

    let markerGraphicsSquare = makeSquare();

    let markerGraphicsCircle = makeSquare();

    const pixiContainer = new PIXI.Container();

    pixiContainer.addChild(markerGraphicsSquare);
    pixiContainer.addChild(markerGraphicsCircle);
    const estiLatLng = [58.82583906333551, 25.807199605129426];

    const pixiOverlay = L.pixiOverlay((utils) => {
      const container = utils.getContainer();
      renderer = utils.getRenderer();
      project = utils.latLngToLayerPoint;
      const markerCoords = project(estiLatLng);
      markerGraphicsCircle.x = markerGraphicsSquare.x = markerCoords.x;
      markerGraphicsCircle.y = markerGraphicsSquare.y = markerCoords.y;

      renderer.render(container);
    }, pixiContainer);

    pixiOverlay.addTo(map);

    // Animation
    startTimeSquare = performance.now();
    startTimeCircle = performance.now();

    let returning = false;

    let squarePathData = generateSquarePath(estiLatLng, 100, [50, 80]);
    let circPath = genCirclePath(estiLatLng, [10, 30], [100, 310]);

    let pathSquare = squarePathData.path;
    let pathCircle = circPath.path;
    function calculateNewSquarePosition(progress) {
      progress = Math.max(0, Math.min(1, progress));

      let index = Math.floor(progress * (pathSquare.length - 1));

      index = Math.max(0, Math.min(pathSquare.length - 2, index));
      const start = pathSquare[index];
      const end = pathSquare[index + 1];

      const segmentProgress = progress * (pathSquare.length - 1) - index;
      const lat = start[0] + (end[0] - start[0]) * segmentProgress;
      const lng = start[1] + (end[1] - start[1]) * segmentProgress;

      return { lat: lat, lng: lng };
    }

    function calculateNewCirclePosition(progress) {
      progress = Math.max(0, Math.min(1, progress));

      let index = Math.floor(progress * (pathCircle.length - 1));

      index = Math.max(0, Math.min(pathCircle.length - 2, index));
      const start = pathCircle[index];
      const end = pathCircle[index + 1];
      const segmentProgress = progress * (pathCircle.length - 1) - index;
      const lat = start[0] + (end[0] - start[0]) * segmentProgress;
      const lng = start[1] + (end[1] - start[1]) * segmentProgress;

      return { lat: lat, lng: lng };
    }

    function animateSquare(time) {
      let progress =
        (time - startTimeSquare) / (squarePathData?.duration || 10000);
      let newPos = calculateNewSquarePosition(progress);
      //animate
      markerGraphicsSquare.clear();

      markerGraphicsSquare = redrawSquare(markerGraphicsSquare);

      let markerCoords = project([newPos.lat, newPos.lng]);
      markerGraphicsSquare.x = markerCoords.x;
      markerGraphicsSquare.y = markerCoords.y;
      if (progress < 1) {
        requestAnimationFrame(animateSquare);
      } else {
        pathSquare.reverse();
        startTimeSquare = performance.now();
        requestAnimationFrame(animateSquare);
      }
      renderer.render(pixiContainer);
    }

    function animateCircle(time) {
      let progress =
        (time - startTimeCircle) / (squarePathData?.duration * 0.1 || 10000);
      let newPos = calculateNewCirclePosition(progress);
      //animate
      markerGraphicsCircle.clear();
      markerGraphicsCircle = redrawCircle(markerGraphicsCircle);
      let markerCoords = project([newPos.lat, newPos.lng]);
      markerGraphicsCircle.x = markerCoords.x;
      markerGraphicsCircle.y = markerCoords.y;
      if (progress < 1) {
        requestAnimationFrame(animateCircle);
      } else {
        markerGraphicsCircle.clear();
      }
      renderer.render(pixiContainer);
    }

    requestAnimationFrame(animateSquare);
    requestAnimationFrame(animateCircle);
  });
</script>

<div id="map" bind:this={mapContainer} />

<style>
  #map {
    height: 100vw;
  }
</style>
