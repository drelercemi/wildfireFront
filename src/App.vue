<script setup>
import { ref, watch, onWatcherCleanup } from 'vue'
import treeImg from './assets/tree.png'
import fireImg from './assets/fire.png'
import ashesImg from './assets/ashes.png'

const simConf = ref({});
const forest = ref([]);
const forestPlotPixelSize = ref(10);
const time = ref(0);
const errorMessage = ref(null);
const playingSim = ref(false);
let clock;

fetch('/api/simulation/configuration')
  .then((response) => response.json())
  .then((json) => {
    simConf.value = json;
    forestPlotPixelSize.value = Math.max(10, Math.floor(1280 / simConf.value.forestWidth));
  });

fetch('/api/forest/0')
  .then((response) => response.json())
  .then((json) => {
    forest.value = json.forest[0].map((col, c) => json.forest.map((row, r) => json.forest[r][c]));
  });

const toImageUrl = (plot) => {
  switch (plot) {
    case 'TREE':
      return treeImg;
    case 'FIRE':
      return fireImg;
    default:
      return ashesImg;
  }
}

watch(time, async (newTime) => {
  const controller = new AbortController();

  console.debug("call http for time " + time.value);
  fetch(`/api/forest/${newTime}`, { signal: controller.signal }).then((response) => response.json())
    .then((json) => {
      forest.value = json.forest[0].map((col, c) => json.forest.map((row, r) => json.forest[r][c]));
    })
    .catch(error => {
      if (error.name === 'AbortError') {
        console.debug(`abortError catched while calling the forest api for time ${newTime}. It most probably comes from user traveling too fast in time.`);
      } else {
        errorMessage.value = error;
        console.error(`error while calling the forest api for time ${newTime}`, error);
      }
    })

  if (time.value == simConf.value.simDuration - 1) {
    clearInterval(clock);
    console.debug("stop simu");
  }

  onWatcherCleanup(() => {
    // abort stale request
    controller.abort();
  })
})

function playPause() {
  if (playingSim.value) {
    console.debug("stop clock");
    clearInterval(clock);
  } else {
    console.debug("run clock");
    clock = setInterval(() => {
      time.value++;
    }, 500);
  }
  playingSim.value = !playingSim.value;
}
//:width="forestPlotPixelSize" :height="forestPlotPixelSize"
</script>

<template>
  <div style="overflow-x:auto;">
    <table>
      <tbody> <!--todo rename c'est pas une colonne-->
        <tr v-for="(forestCol, y) in forest" :key="y">
          <td v-for="(forestPlot, x) in forestCol" :key="x"><img :src="toImageUrl(forestPlot)"
              :alt="forestPlot.toLowerCase()"></td>
        </tr>
      </tbody>
    </table>
  </div>
  <div>
    <button @click="playPause()"> {{ playingSim ? "Pause" : "Play" }}
    </button>
    sim time: <input type="range" v-model="time" min="0" :max="simConf.simDuration - 1">
    <!-- {{ time }}s -->
  </div>

  <p>simulation initial parameters</p>

  <ul>
    <li>width: {{ simConf.forestWidth }}</li>
    <li>Height: {{ simConf.forestHeight }}</li>
    <li>Propagation rate: {{ simConf.firePropagationRate }}%</li>
    <li>Burning trees at t=0: {{ simConf.firstBurningTrees }}</li>
  </ul>

</template>

<style scoped>
table {
  background-color: blanchedalmond;
  border-spacing: 0;
  line-height: 0;
}

table,
th,
td {
  border: 0px;
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
}

td,
img {

  min-width: 20px;
  min-height: 20px;
}

.tree-image-cell {
  background-image: src="treeImg";
}

.fire-image-cell {
  background-image: src="fireImg";
}

.ashes-image-cell {
  background-image: src="ashesImg";
}
</style>
