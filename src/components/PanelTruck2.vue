<template>
  <div class="panelContainer">
    <div v-if="inError" id="error">
      <font-awesome-icon icon="exclamation-triangle"></font-awesome-icon>
      Sorry, something went wrong.
    </div>

    <figure v-else>
      <div id="ol-wrapper" ref="olWrapper"></div>

      <div
        id="caption-wrapper"
        ref="captionWrapper"
        v-if="validated && captionShown"
      >
        <div id="caption-left-control" class="caption-control">
          <div v-if="!(currentScene < 0)" @click="changeScene('back')">
            <font-awesome-icon icon="arrow-left"></font-awesome-icon>
          </div>
        </div>
        <div id="caption-center">
          <figcaption v-html="captionCenter"></figcaption>
          <div
            v-if="currentScene >= 0 && currentScene < screenplay.scenes.length"
            id="caption-bottom-control"
          >
            <span class="counter"
              >{{ currentScene + 1 }}/{{ screenplay.scenes.length }}</span
            >
            <font-awesome-icon
              class="control-bottom-icon"
              icon="eye-slash"
              @click="captionShown = false"
            ></font-awesome-icon>
            <font-awesome-icon
              class="control-bottom-icon"
              icon="external-link-alt"
              v-if="contextUrl.length > 0"
              @click="openContextUrl"
            ></font-awesome-icon>
            <font-awesome-icon
              :icon="fullScreenIcon"
              class="control-bottom-icon"
              @click="toggleFullscreen"
            ></font-awesome-icon>
          </div>
          <div
            class="begin-callout"
            v-if="currentScene === -1"
            @click="changeScene(0)"
          >
            Begin
          </div>
        </div>
        <div id="caption-right-control" class="caption-control">
          <div @click="changeScene('forward')">
            <font-awesome-icon
              :icon="
                currentScene >= screenplay.scenes.length
                  ? 'undo-alt'
                  : 'arrow-right'
              "
            ></font-awesome-icon>
          </div>
        </div>
      </div>
      <div id="caption-shower" v-if="!captionShown">
        <font-awesome-icon
          icon="eye"
          @click="captionShown = true"
          class="control-bottom-icon"
        ></font-awesome-icon>
      </div>
    </figure>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";
import {
  library as faLibrary,
  config as faConfig,
} from "@fortawesome/fontawesome-svg-core";
import {
  faExclamationTriangle,
  faArrowRight,
  faArrowLeft,
  faEye,
  faEyeSlash,
  faUndoAlt,
  faExternalLinkAlt,
  faExpandArrowsAlt,
  faCompressArrowsAlt,
} from "@fortawesome/free-solid-svg-icons";
import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome";

faConfig.autoAddCss = false;
faLibrary.add(
  faExclamationTriangle,
  faArrowRight,
  faArrowLeft,
  faEye,
  faEyeSlash,
  faUndoAlt,
  faExternalLinkAlt,
  faExpandArrowsAlt,
  faCompressArrowsAlt
);

import { Map, View } from "ol";
import TileLayer from "ol/layer/Tile";
import ImageLayer from "ol/layer/Image";
import IIIF from "ol/source/IIIF";
import IIIFInfo from "ol/format/IIIFInfo";
import OSM from "ol/source/OSM";
import Static from "ol/source/ImageStatic";
import XYZ from "ol/source/XYZ";
import TileJSON from "ol/source/TileJSON";
import { marked } from "marked";

const imageMeasurer = async (src) => {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => resolve([img.height, img.width]);
    img.onerror = reject;
    img.src = src;
  });
};

const props = defineProps({
  screenplaySrc: String,
});

let inError = ref(false);
let validated = ref(false);
//let errorMessage = ref("");
let screenplay = ref({});
let currentScene = ref(-1);
let layers = ref({});
let ol = ref(null);
let captionShown = ref(true);
let currentMaxExtent = ref([]);
let fullScreenIcon = ref("expand-arrows-alt");
let olWrapper = ref(null);
let captionWrapper = ref(null);

let captionCenter = computed(() => {
  const currentSceneValue = currentScene.value;
  const { metadata, scenes } = screenplay.value;

  if (currentSceneValue < 0) {
    const { title, subtitle, author } = metadata;
    return `<h1 class="title">${title}${
      subtitle ? `<h2 class="subtitle">${subtitle}</h2>` : ""
    }${author ? `<h3 class="author">by ${author}</h3>` : ""}</h1>`;
  } else if (currentSceneValue >= scenes.length) {
    const { title, author, publishedDate } = metadata;
    let caption = `<p><strong>${title}</strong></p>`;
    if (author) caption += `<p>by ${author}</p>`;
    if (publishedDate) caption += `<p>published on ${publishedDate}</p>`;
    caption += `<p class="small-credit">Created with panel-truck 0.2, <a href="https://leventhalmap.org">Leventhal Map and Education Center</a> at the Boston Public Library</p>`;
    return caption;
  } else {
    const thisScene = scenes[currentSceneValue];
    const {
      caption: { title, text },
    } = thisScene;
    let caption = "";
    if (title && title.length > 0) {
      caption += `<h4 class="caption-title">${marked.parse(title)}</h4>`;
    }
    caption += marked.parse(text);
    return caption;
  }
});

const contextUrl = computed(() => {
  if (
    !validated.value ||
    currentScene.value < 0 ||
    currentScene.value >= screenplay.value.scenes.length
  ) {
    return "";
  }

  const currentSceneData = screenplay.value.scenes[currentScene.value];
  const source = currentSceneData.moreInfo
    ? currentSceneData.moreInfo
    : screenplay.value.sources.find((d) => d.id === currentSceneData.source)
        ?.moreInfo || "";

  return source;
});

function toggleFullscreen() {
  const elem = document.getElementById("app");
  const doc = document;
  const docElm = document.documentElement;

  const fullscreenElement =
    doc.fullscreenElement ||
    doc.webkitFullscreenElement ||
    doc.mozFullScreenElement ||
    doc.msFullscreenElement;

  if (!fullscreenElement) {
    if (elem.requestFullscreen) {
      elem.requestFullscreen();
    } else if (elem.webkitRequestFullscreen) {
      elem.webkitRequestFullscreen();
    } else if (elem.msRequestFullscreen) {
      elem.msRequestFullscreen();
    } else if (docElm.mozRequestFullScreen) {
      docElm.mozRequestFullScreen();
    }

    fullScreenIcon.value = "compress-arrows-alt";
  } else {
    if (doc.exitFullscreen) {
      doc.exitFullscreen();
    } else if (doc.webkitExitFullscreen) {
      doc.webkitExitFullscreen();
    } else if (doc.mozCancelFullScreen) {
      doc.mozCancelFullScreen();
    } else if (doc.msExitFullscreen) {
      doc.msExitFullscreen();
    }
  }
}

async function validateScreenplay() {
  try {
    const response = await fetch(props.screenplaySrc);
    const screenplayData = await response.json();

    if (
      typeof screenplayData.metadata === "object" &&
      Array.isArray(screenplayData.sources) &&
      Array.isArray(screenplayData.scenes)
    ) {
      inError.value = false;
      validated.value = true;
      screenplay.value = screenplayData;
      return Promise.resolve();
    } else {
      return Promise.reject();
    }
  } catch (error) {
    return Promise.reject(error);
  }
}

function changeScene(to, initial) {
  const currentSceneValue = currentScene.value;
  const scenesLength = screenplay.value.scenes.length;

  let nextSceneNumber;

  if (to === "forward") {
    nextSceneNumber = currentSceneValue + 1;
  } else if (to === "back") {
    nextSceneNumber = currentSceneValue - 1;
  } else if (typeof to === "number") {
    nextSceneNumber = to;
  } else {
    return;
  }

  nextSceneNumber = Math.min(scenesLength, Math.max(-1, nextSceneNumber));

  let nextFunctionalScene =
    nextSceneNumber === -1
      ? 0
      : nextSceneNumber === scenesLength
      ? scenesLength - 1
      : nextSceneNumber;

  let currentSceneSource =
    currentSceneValue === -1
      ? screenplay.value.scenes[0].source
      : currentSceneValue === scenesLength
      ? screenplay.value.scenes[currentSceneValue - 1].source
      : screenplay.value.scenes[currentSceneValue].source;

  currentScene.value = nextSceneNumber;

  const nextScene = screenplay.value.scenes[nextFunctionalScene];
  const nextSceneSource = nextScene.source;

  if (initial || currentSceneSource !== nextSceneSource) {
    changeSource(nextFunctionalScene)
      .then(() => changeExtent(nextFunctionalScene))
      .catch(() => {
        inError.value = true;
      });
  } else {
    changeExtent(nextFunctionalScene, 3000);
  }
}

async function changeSource(sceneNumber) {
  try {
    const scenes = screenplay.value.scenes;
    const sources = screenplay.value.sources;

    const sceneSource = scenes[sceneNumber].source;
    const newSource = sources.find((d) => d.id === sceneSource);

    if (!newSource) {
      throw new Error("Source not found");
    }

    if (newSource.type.toLowerCase() === "iiifmanifest") {
      const sequence = newSource.sequence || 0;
      const canvas = newSource.canvas || 0;
      const image = newSource.image || 0;

      const manifestResponse = await fetch(newSource.manifest);
      const manifest = await manifestResponse.json();

      const imageEndpoint =
        manifest?.sequences[sequence]?.canvases[canvas]?.images[image]?.resource
          ?.service?.["@id"];
      if (imageEndpoint) {
        return loadNewIIIF(imageEndpoint);
      }
    } else if (newSource.type.toLowerCase() === "iiifimage") {
      return loadNewIIIF(newSource.imageEndpoint);
    } else if (newSource.type.toLowerCase() === "staticimage") {
      return loadNewImage(newSource.imageFile);
    } else if (newSource.type.toLowerCase() === "geomap") {
      return loadNewGeoMap(newSource);
    }

    throw new Error("Unsupported source type");
  } catch (error) {
    throw error;
  }
}

function changeExtent(sceneNumber, transitionDuration) {
  const scenes = screenplay.value.scenes;
  const olView = ol.value.getView();
  const currentMaxExtentValue = currentMaxExtent.value;
  const captionWrapperHeight = captionWrapper.value.clientHeight;

  const extent = scenes[sceneNumber].extent || "fit";
  const bottomPadding =
    captionWrapperHeight > 50 ? captionWrapperHeight + 20 : 70;

  const fitOptions = {
    padding: [10, 10, bottomPadding, 10],
    duration:
      typeof transitionDuration === "number" ? transitionDuration : undefined,
  };

  const viewToFit = extent === "fit" ? currentMaxExtentValue : extent;

  olView.fit(viewToFit, fitOptions);
}

async function loadNewIIIF(imageEndpoint) {
  try {
    const response = await fetch(imageEndpoint);
    const iiif = await response.json();

    const options = new IIIFInfo(iiif).getTileSourceOptions();
    if (options === undefined || options.version === undefined) {
      return;
    }

    options.zDirection = -1;
    const iiifTileSource = new IIIF(options);
    currentMaxExtent.value = iiifTileSource.getTileGrid().getExtent();

    layers.value.iiif.setSource(iiifTileSource);
    layers.value.iiif.setVisible(true);
    layers.value.geoMap.setVisible(false);
    layers.value.staticImage.setVisible(false);

    return Promise.resolve();
  } catch (error) {
    console.log(imageEndpoint);
    inError.value = true;
    return Promise.reject(error);
  }
}

async function loadNewImage(imageSrc) {
  try {
    const hw = await imageMeasurer(imageSrc);

    layers.value.staticImage.setSource(
      new Static({
        url: imageSrc,
        imageExtent: [0, 0, hw[1], hw[0]],
      })
    );

    currentMaxExtent.value = [0, 0, hw[1], hw[0]];
    layers.value.staticImage.setVisible(true);
    layers.value.iiif.setVisible(false);
    layers.value.geoMap.setVisible(false);

    return Promise.resolve();
  } catch (error) {
    inError.value = true;
    return Promise.reject(error);
  }
}
async function loadNewGeoMap(mapSrc) {
  try {
    let mapSource;
    if (mapSrc.tileXYZ) {
      mapSource = new XYZ({ url: mapSrc.tileXYZ });
    } else if (mapSrc.tileJSON) {
      mapSource = new TileJSON({ url: mapSrc.tileJSON });
    } else {
      mapSource = new OSM();
    }

    layers.value.geoMap.setSource(mapSource);
    currentMaxExtent.value = [
      -20026376.39, -20048966.1, 20026376.39, 20048966.1,
    ];
    layers.value.geoMap.setVisible(true);
    layers.value.staticImage.setVisible(false);
    layers.value.iiif.setVisible(false);

    return Promise.resolve();
  } catch (error) {
    return Promise.reject(error);
  }
}

function openContextUrl() {
  window.open(contextUrl.value);
}

onMounted(async () => {
  layers.value.iiif = new TileLayer({ visible: false });
  layers.value.geoMap = new TileLayer({ visible: false });
  layers.value.staticImage = new ImageLayer({ visible: false });

  const wrapperDom = olWrapper.value;

  const waitForDom = () => {
    return new Promise((resolve) => {
      const checkInterval = setInterval(() => {
        if (wrapperDom.clientHeight > 0) {
          clearInterval(checkInterval);
          resolve();
        }
      }, 100);
    });
  };

  try {
    await waitForDom();
    ol.value = new Map({
      target: olWrapper.value,
      layers: [
        layers.value.iiif,
        layers.value.geoMap,
        layers.value.staticImage,
      ],
      view: new View({
        center: [0, 0],
        zoom: 0,
      }),
    });

    await validateScreenplay();
    changeScene(-1, true);
  } catch (error) {
    console.error("Error while waiting for DOM or initializing:", error);
    inError.value = true;
  }
});
</script>

<style scoped>
#error {
  text-align: center;
  padding-top: 1em;
}

#panelContainer {
  width: 100%;
  height: 100%;
  background-color: #dcdcdc;
  position: relative;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
    Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
}

figure {
  margin: 0;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
    Oxygen, Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif !important;
}

.hidden {
  display: none;
}

/* Grouped #ol-wrapper, #caption-wrapper, #caption-shower */
#ol-wrapper,
#caption-wrapper,
#caption-shower {
  position: absolute;
}

#ol-wrapper {
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 1;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica,
    Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol";
}

#caption-wrapper {
  bottom: 10px;
  left: 10px;
  right: 10px;
  z-index: 2;
  display: flex;
  flex-direction: row;
  background-color: rgba(253, 249, 243, 0.96);
  border-radius: 3px;
}

#caption-shower {
  bottom: 0;
  left: 40px;
  width: auto;
  padding: 10px 10px 5px;
  background-color: rgba(253, 249, 243, 0.96);
  font-size: 0.85rem;
  color: rgb(163, 158, 148);
  z-index: 2;
}

.begin-callout {
  font-weight: bold;
  font-size: 1.2rem;
  text-align: right;
  cursor: pointer;
}

.begin-callout:hover {
  color: rgb(104, 18, 18);
}

.caption-control {
  min-width: 2em;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
}

.caption-control > div {
  display: inline-block;
  cursor: pointer;
  font-size: 1.2rem;
  transition: color 0.4s;
}

.caption-control > div:hover {
  color: rgb(104, 18, 18);
}

#caption-left-control {
  border-right: 2px dotted rgba(238, 232, 221, 0.99);
}

#caption-right-control {
  border-left: 2px dotted rgba(238, 232, 221, 0.99);
}

#caption-left-control > div {
  position: absolute;
  left: 5px;
  bottom: 10px;
  text-align: left;
}

#caption-right-control > div {
  position: absolute;
  right: 5px;
  bottom: 10px;
  text-align: right;
}

#caption-center {
  flex-grow: 1;
  padding: 20px 20px 10px;
}

#caption-center a {
  color: black;
  text-decoration: none;
  background-image: linear-gradient(
    0deg,
    rgba(163, 158, 148, 0.7) 39%,
    rgba(0, 0, 0, 0) 0px
  );
}

#caption-center a:hover {
  background-image: linear-gradient(
    0deg,
    rgba(104, 18, 18, 0.3) 39%,
    rgba(0, 0, 0, 0) 0px
  );
}

#caption-bottom-control {
  font-size: 0.85rem;
  color: rgb(163, 158, 148);
}

.control-bottom-icon {
  transition: color 0.4s;
}

.control-bottom-icon:not(:last-of-type) {
  margin-right: 7px;
}

.control-bottom-icon:hover {
  cursor: pointer;
  color: rgb(104, 18, 18);
}

h1.title {
  margin: 0 0 10px 0;
  font-weight: 700;
  text-shadow: 2px 2px 0px rgba(131, 126, 118, 0.3);
}

h2.subtitle {
  margin: 0 0 8px 0;
  font-size: 1.3rem;
  font-weight: 400;
}

h3.author {
  margin: 0;
  font-size: 1rem;
  font-weight: 400;
}

.caption-title {
  margin: 0 0 5px 0;
  font-size: 1.2rem;
}

#caption-center p {
  margin: 0 0 5px 0;
}

.counter {
  margin: 0 10px 0 0;
  font-weight: 700;
  font-size: 0.8rem;
}

#caption-center p.small-credit {
  font-size: 0.9rem;
  margin: 15px 0 0 0;
  color: rgb(163, 158, 148);
}

.ol-attribution.ol-uncollapsible {
  top: 0;
  right: 0;
  bottom: unset;
  border-radius: 0;
  font-size: 0.7rem;
}

.ol-wrapper-faded {
  opacity: 0.2;
}
</style>
