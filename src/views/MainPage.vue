<template>
  <div id="google-canvas"></div>
</template>

<script lang="ts" setup>
/* eslint-disable no-undef */
import * as THREE from "three";
import { ThreeJSOverlayView } from "@googlemaps/three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";

import { Loader } from "@googlemaps/js-api-loader";
import { onMounted, ref } from "vue";

const GOOGLE_MAPS_API_KEY = "AIzaSyAgT1N3rDEz5gsjmO-XUcSpY6261SdS5FE";
const mapLoader = new Loader({ apiKey: GOOGLE_MAPS_API_KEY });
const loaderPromise = mapLoader.load();

onMounted(async () => {
  await loaderPromise;

  const mainCanvas = document.getElementById("google-canvas")!;

  const mapOptions = {
    tilt: 0,
    heading: 0,
    zoom: 18,
    center: { lat: 35.6594945, lng: 139.6999859 },
    mapId: "15431d2b469f209e",
    disableDefaultUI: true,
    gestureHandling: "none",
    keyboardShortcuts: false,
  };

  const map = new google.maps.Map(mainCanvas, mapOptions);
  const webglOverlayView = new google.maps.WebGLOverlayView();

  var scene: THREE.Scene;
  var camera: THREE.PerspectiveCamera;
  var loader: GLTFLoader;
  var renderer: THREE.WebGLRenderer;

  webglOverlayView.onAdd = () => {
    // Set up the scene.

    scene = new THREE.Scene();

    camera = new THREE.PerspectiveCamera();

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.75); // Soft white light.
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.25);
    directionalLight.position.set(0.5, -1, 0.5);
    scene.add(directionalLight);

    // Load the model.
    loader = new GLTFLoader();
    const source =
      "https://raw.githubusercontent.com/googlemaps/js-samples/main/assets/pin.gltf";
    loader.load(source, (gltf) => {
      gltf.scene.scale.set(10, 10, 10);
      gltf.scene.rotation.x = Math.PI; // Rotations are in radians.
      scene.add(gltf.scene);
    });
  };

  webglOverlayView.onContextRestored = ({ gl }) => {
    // Create the js renderer, using the
    // maps's WebGL rendering context.
    renderer = new THREE.WebGLRenderer({
      canvas: gl.canvas,
      context: gl,
      ...gl.getContextAttributes(),
    });
    renderer.autoClear = false;

    // Wait to move the camera until the 3D model loads.
    loader.manager.onLoad = () => {
      renderer.setAnimationLoop(() => {
        webglOverlayView.requestRedraw();
        const { tilt, heading, zoom } = mapOptions;
        map.moveCamera({ tilt, heading, zoom });

        // Rotate the map 360 degrees.
        if (mapOptions.tilt < 67.5) {
          mapOptions.tilt += 0.5;
        } else if (mapOptions.heading <= 360) {
          mapOptions.heading += 0.2;
          mapOptions.zoom -= 0.0005;
        } else {
          renderer.setAnimationLoop(null);
        }
      });
    };
  };

  webglOverlayView.onDraw = ({ gl, transformer }): void => {
    const latLngAltitudeLiteral: google.maps.LatLngAltitudeLiteral = {
      lat: mapOptions.center.lat,
      lng: mapOptions.center.lng,
      altitude: 100,
    };

    // Update camera matrix to ensure the model is georeferenced correctly on the map.
    const matrix = transformer.fromLatLngAltitude(latLngAltitudeLiteral);
    camera.projectionMatrix = new THREE.Matrix4().fromArray(matrix);

    webglOverlayView.requestRedraw();
    renderer.render(scene, camera);

    // Sometimes it is necessary to reset the GL state.
    renderer.resetState();
  };
  webglOverlayView.setMap(map);
});
</script>

<style>
#street-canvas {
  width: 80%;
  height: 70%;
}
</style>
