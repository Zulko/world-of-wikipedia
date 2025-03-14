<script>
  import { onMount, onDestroy } from "svelte";
  import { createEventDispatcher } from "svelte";
  import L from "leaflet";
  import "leaflet/dist/leaflet.css";

  const basePath = import.meta.env.BASE_URL;
  const dispatch = createEventDispatcher();

  // ===== PROPS =====
  export let markers = [];
  export let targetLocation = null; // Format: { lat, lon, zoom }

  // ===== STATE VARIABLES =====
  let mapElement;
  let map;
  let markerLayer = null;
  let isFlying = false;
  let hoveredMarkerId = null;
  let existingMapMarkers = new Map(); // key: marker.id, value: L.marker object
  let resizeObserver;
  let boundsChangeTimeout = null;

  // ===== LIFECYCLE METHODS =====
  onMount(() => {
    initializeMap();
    updateMarkers();

    // Initialize ResizeObserver
    resizeObserver = new ResizeObserver(() => {
      if (map) map.invalidateSize();
    });
    resizeObserver.observe(mapElement);
  });

  onDestroy(() => {
    if (map) {
      map.off("moveend", handleBoundsChange);
      map.off("zoomend", handleBoundsChange);
      map.off("resize", handleBoundsChange);
      map.remove();
      map = null;
    }

    if (resizeObserver) {
      resizeObserver.disconnect();
    }
  });

  // ===== REACTIVE DECLARATIONS =====
  $: if (map && targetLocation) {
    updateMarkers();
    flyTo(targetLocation);
  }

  $: if (markers && map) {
    updateMarkers();
  }

  // ===== MAP INITIALIZATION =====
  function initializeMap() {
    // Initialize the map
    map = L.map(mapElement, {
      zoomControl: false,
      worldCopyJump: true,
    }).setView([0, 0], 3);

    if (targetLocation) {
      flyTo(targetLocation);
    }

    // Add tile layer (OpenStreetMap)
    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution:
        '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    }).addTo(map);

    // Add zoom control to bottom right
    L.control
      .zoom({
        position: "bottomright",
      })
      .addTo(map);

    // Create layer groups for different marker types
    const markerLayers = [
      { name: "dot", zIndex: 200 },
      { name: "reduced", zIndex: 300 },
      { name: "full", zIndex: 400 },
      { name: "selected", zIndex: 500 },
    ];

    markerLayers.forEach((layer) => {
      map.createPane(layer.name);
      map.getPane(layer.name).style.zIndex = layer.zIndex;
    });

    markerLayer = L.layerGroup().addTo(map);

    // Add event listeners
    map.on("moveend", handleBoundsChange);
    map.on("zoomend", handleBoundsChange);
    map.on("resize", handleBoundsChange);
  }

  // ===== MAP CONTROL FUNCTIONS =====
  function flyTo(targetLocation) {
    const { lat, lon, zoom: targetZoom } = targetLocation;

    // If target zoom is less than current zoom, clear markers first
    if (targetZoom < map.getZoom()) {
      markerLayer.clearLayers();
    }

    isFlying = true;
    map.flyTo([lat, lon], targetZoom, {
      animate: true,
      duration: 1, // Duration in seconds
    });

    // Set isFlying back to false after animation completes
    setTimeout(() => {
      isFlying = false;
      // Dispatch a single boundschange event after flying completes
      handleBoundsChange();
    }, 1100); // Slightly longer than animation duration

    targetLocation = null;
  }

  // ===== EVENT HANDLERS =====
  function handleBoundsChange() {
    if (!map || isFlying) return;

    const bounds = map.getBounds();

    // Debounce the dispatch to avoid too frequent updates
    clearTimeout(boundsChangeTimeout);
    boundsChangeTimeout = setTimeout(() => {
      dispatch("boundschange", {
        bounds: bounds,
        center: bounds.getCenter(),
        zoom: map.getZoom(),
      });
    }, 200);
  }

  function handleResize() {
    if (map) {
      // Use a small timeout to ensure the resize happens after CSS transitions complete
      setTimeout(() => {
        map.invalidateSize({ animate: true });
        // Trigger a bounds change to update markers if needed
        handleBoundsChange();
      }, 300); // Match this with your CSS transition duration
    }
  }

  // ===== MARKER MANAGEMENT =====
  function computeMarkerHtml(marker) {
    const iconByType = {
      adm1st: "map",
      adm2nd: "map",
      adm3rd: "map",
      airport: "plane-takeoff",
      building: "building",
      church: "church",
      city: "city",
      country: "flag",
      county: "map",
      edu: "school",
      event: "newspaper",
      forest: "trees",
      glacier: "mountain-snow",
      island: "tree-palm",
      isle: "tree-palm",
      landmark: "landmark",
      locality: "locality",
      mountain: "mountain-snow",
      other: "pin",
      railwaystation: "train-front",
      river: "waves",
      school: "school",
      settlement: "city",
      town: "city",
      village: "city",
      waterbody: "waves",
    };

    const icon = iconByType[marker.category] || iconByType.other;
    const unsanitized_page_title = marker.page_title.replaceAll("_", " ");

    let label = marker.name;
    if (!marker.name) {
      label = unsanitized_page_title;
    } else if (unsanitized_page_title.includes(marker.name)) {
      label = unsanitized_page_title;
    } else {
      const fullLabel = marker.name + " - " + unsanitized_page_title;
      if (fullLabel.length <= 30) {
        label = fullLabel;
      }
    }

    return `
    <div class="map-marker marker-display-${marker.displayClass}" >
        <div class="marker-icon-circle">
          <img src="${basePath}icons/${icon}.svg">
        </div>
        <div class="marker-text-container">
          <div class="marker-text marker-text-outline">${label}</div>
          <div class="marker-text">${label}</div>
        </div>
      </div>
    `;
  }

  function makeIcon(marker, displayClass) {
    const markerHtml = computeMarkerHtml(marker);

    const iconSizesByDisplayClass = {
      dot: [18, 18],
      reduced: [28, 28],
      full: [128, 32],
      selected: [128, 32],
    };

    return L.divIcon({
      className: "custom-div-icon",
      html: markerHtml,
      iconSize: iconSizesByDisplayClass[displayClass],
    });
  }

  function updateMarkers() {
    if (!map || !markerLayer) return;

    // Track which markers we've processed to identify removals
    const processedIds = new Set();

    // Update or add markers
    markers.forEach((marker) => {
      processedIds.add(marker.id);

      let displayClass = marker.displayClass;
      let pane = marker.displayClass;

      if (hoveredMarkerId === marker.id && displayClass !== "selected") {
        displayClass = "full";
        pane = "selected";
      }

      // Check if marker already exists
      if (existingMapMarkers.has(marker.id)) {
        updateExistingMarker(marker, displayClass, pane);
      } else {
        createNewMarker(marker, displayClass, pane);
      }
    });

    // Remove markers that are no longer in the data
    removeStaleMarkers(processedIds);
  }

  function updateExistingMarker(marker, displayClass, pane) {
    const { existingMarker, existingClass } = existingMapMarkers.get(marker.id);

    // Update position if needed
    if (
      existingMarker._latlng.lat !== marker.lat ||
      existingMarker._latlng.lng !== marker.lon
    ) {
      existingMarker.setLatLng([marker.lat, marker.lon]);
    }

    // Update pane if needed
    if (existingMarker.options.pane !== pane) {
      existingMarker.options.pane = pane;
      // Force marker to use the new pane
      existingMarker.removeFrom(map);
      existingMarker.addTo(map);
    }

    // Update icon if display class changed
    if (existingClass !== displayClass) {
      const icon = makeIcon(marker, displayClass);
      existingMarker.setIcon(icon);

      // Update the stored display class
      existingMapMarkers.set(marker.id, {
        existingMarker,
        displayClass,
      });
    }
  }

  function createNewMarker(marker, displayClass, pane) {
    const icon = makeIcon(marker, displayClass);
    const mapMarker = L.marker([marker.lat, marker.lon], {
      icon: icon,
      pane: pane,
    });

    // Add event handlers
    mapMarker.on("click", () => {
      dispatch("markerclick", marker);
    });

    mapMarker.on("mouseover", () => {
      if (hoveredMarkerId !== marker.id) {
        hoveredMarkerId = marker.id;
        updateMarkers();
      }
    });

    // Store reference to the new marker
    existingMapMarkers.set(marker.id, {
      existingMarker: mapMarker,
      displayClass: displayClass,
    });

    // Add marker to the map
    mapMarker.addTo(markerLayer);
  }

  function removeStaleMarkers(processedIds) {
    for (const [markerId, entry] of existingMapMarkers.entries()) {
      if (!processedIds.has(markerId)) {
        markerLayer.removeLayer(entry.existingMarker);
        existingMapMarkers.delete(markerId);
      }
    }
  }

  // ===== EXPORTED FUNCTIONS =====
  export function invalidateMapSize() {
    handleResize();
  }
</script>

<div
  class="map-container"
  bind:this={mapElement}
  on:resize={handleResize}
></div>

<style>
  .map-container {
    width: 100%;
    height: 100%;
  }

  :global(.leaflet-container) {
    height: 100%;
    width: 100%;
  }

  :global(.map-marker) {
    transition:
      transform 0.1s ease,
      filter 0.1s ease;
    display: block;
    text-align: center;

    &:hover {
      transform: scale(1.1);
      filter: brightness(1.2);
      z-index: 1000 !important; /* Ensure hovered markers appear above others */
    }
    &:hover > .marker-icon-circle,
    &.marker-display-selected > .marker-icon-circle {
      --circle-size: 32px !important;
      box-shadow: 4px 4px 8px rgba(0, 0, 0, 0.35);
      z-index: 1000 !important;
    }

    &.marker-display-selected {
      /* Ensure hovered markers appear above others */
      z-index: 1000 !important;
    }

    &:hover > .marker-text-container,
    &.marker-display-selected > .marker-text-container {
      visibility: visible;
      opacity: 1;
    }

    &.marker-display-selected > .marker-icon-circle {
      border: 6px solid #f00707;
      box-shadow: 8px 8px 16px rgba(0, 0, 0, 0.95);
    }

    &.marker-display-full > .marker-text-container {
      visibility: visible;
      opacity: 1;
    }

    &.marker-display-full > .marker-icon-circle {
      --circle-size: 32px;
      box-shadow: 4px 4px 8px rgba(0, 0, 0, 0.35);
    }

    &.marker-display-reduced > .marker-icon-circle {
      --circle-size: 24px;
    }

    &.marker-display-dot > .marker-icon-circle {
      --circle-size: 18px;
    }
    &.marker-display-dot > .marker-icon-circle > img {
      width: calc(var(--circle-size) * 0.8);
      height: calc(var(--circle-size) * 0.8);
    }

    & > .marker-icon-circle {
      width: var(--circle-size);
      height: var(--circle-size);
      background-color: white;
      border-radius: 50%;
      border: 2px solid #222;
      display: flex;
      align-items: center;
      justify-content: center;
      margin: 0 auto; /* Center the circle in its parent div */
      transition:
        width 0.1s ease,
        height 0.1s ease;
    }
    & > .marker-icon-circle img {
      width: calc(var(--circle-size) * 0.7);
      height: calc(var(--circle-size) * 0.7);
      margin: 0; /* Remove bottom margin to help with centering */
      display: block; /* Ensure the image behaves as a block */
      transition:
        width 0.1s ease,
        height 0.1s ease;
    }
  }

  :global(.marker-text-container) {
    margin-top: -5px;
    font-size: 14px;
    line-height: 0.9em;
    position: relative;
    text-align: center;
    visibility: hidden;
    opacity: 0;
    transition:
      visibility 0.1s ease,
      opacity 0.1s ease;
    & > .marker-text-outline {
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      font-weight: bold;
      color: white;
      -webkit-text-stroke: 6px white;
      text-stroke: 6px white;
      z-index: 1;
    }
  }

  :global(.marker-text) {
    font-weight: bold;
    color: #111;
    position: relative;
    z-index: 2;
  }
  :global(.marker-text-outline) {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    font-weight: bold;
    color: white;
    -webkit-text-stroke: 6px white;
    text-stroke: 6px white;
    -webkit-text-stroke-linejoin: round;
    text-stroke-linejoin: round;
    z-index: 1;
  }
</style>
