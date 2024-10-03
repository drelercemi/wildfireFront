<script setup>
import { ref, watch, onWatcherCleanup, computed } from 'vue'
import treeImg from './assets/tree.png'
import fireImg from './assets/fire.png'
import ashesImg from './assets/ashes.png'
import playImg from './assets/play.svg'
import pauseImg from './assets/pause.svg'
import infoImg from './assets/info.svg'

const simConf = ref({});
const forest = ref([]);
const time = ref(0);
const errorMessage = ref(null);
const playingSim = ref(false);
let clock;

fetch('/api/simulation/configuration')
  .then((response) => response.json())
  .then((json) => {
    simConf.value = json;
    errorMessage.value = null;
  }).catch(error => {
    errorMessage.value = error.message;
    console.error("error while initial conf api call", error);
  });

fetch('/api/forest/0')
  .then((response) => response.json())
  .then((json) => {
    forest.value = json.forest[0].map((col, c) => json.forest.map((row, r) => json.forest[r][c]));
    errorMessage.value = null;
  }).catch(error => {
    errorMessage.value = error.message;
    console.error("error while initial forest api call (t=0)", error);
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

const buttonImg = computed(() => {
  return playingSim.value ? pauseImg : playImg;
});

watch(time, async (newTime) => {
  const controller = new AbortController();

  console.debug("call http for time " + time.value);
  fetch(`/api/forest/${newTime}`, { signal: controller.signal }).then((response) => response.json())
    .then((json) => {
      forest.value = json.forest[0].map((col, c) => json.forest.map((row, r) => json.forest[r][c]));
      errorMessage.value = null;
    })
    .catch(error => {
      if (error.name === 'AbortError') {
        console.debug(`abortError catched while calling the forest api for time ${newTime}. It most probably comes from user traveling too fast in time.`);
      } else {
        errorMessage.value = error.message;
        console.error(`error while calling the forest api for time ${newTime}`, error);
      }
    })

  if (time.value == simConf.value.simDuration - 1) {
    stopSim();
  }

  onWatcherCleanup(() => {
    // abort stale request
    controller.abort();
  })
})

function startSim() {
  console.debug("run clock");
  clock = setInterval(() => {
    time.value++;
  }, 500);
  playingSim.value = true;
}

function stopSim() {
  console.debug("stop clock");
  clearInterval(clock);
  playingSim.value = false;
}

function playPause() {
  if (playingSim.value) {
    stopSim();
  } else {
    if (time.value >= simConf.value.simDuration - 1) {
      time.value = 0;
    }
    startSim();
  }
}

</script>

<template>

  <div v-if="errorMessage" class="error">
    <div class="errorMsg">{{ errorMessage }}</div>
  </div>

  <div>
    <table>
      <tbody>
        <tr v-for="(forestRow, y) in forest" :key="y">
          <td v-for="(forestPlot, x) in forestRow" :key="x">
            <div class="image">
              <img width="100" height="100" :src="toImageUrl(forestPlot)" :alt="forestPlot.toLowerCase()">
            </div>
          </td>
        </tr>
      </tbody>
    </table>

    <div class="player">
      <div class="corner">
        <img class="command" :src="buttonImg" @click="playPause()" :alt="`playingSim ? 'Pause':'Play'`">
      </div>
      <div class="middle">
        <input class="command" type="range" v-model="time" min="0" :max="simConf.simDuration - 1">
      </div>
      <div class="tooltip corner">
        <img class="command" :src="infoImg" alt="info tooltip">
        <span class="tooltiptext">
          <p>simulation initial parameters</p>
          <ul>
            <li>width: {{ simConf.forestWidth }}</li>
            <li>Height: {{ simConf.forestHeight }}</li>
            <li>Propagation rate: {{ simConf.firePropagationRate }}%</li>
            <li>Burning trees at t=0: {{ simConf.firstBurningTrees }}</li>
            <li>Simulation duration: {{ simConf.simDuration - 1 }}</li>
            <li>Current time: {{ time }}</li>
          </ul>
        </span>
      </div>
    </div>
  </div>

</template>

<style scoped>
table {
  background-color: blanchedalmond;
  padding: 10px;
  border-spacing: 0;
  line-height: 0;
}

th,
td {
  padding: 1px;
  vertical-align: top;
  text-align: center;
}

td,
img {
  min-width: 20px;
  min-height: 20px;
}

.image img {
  width: 100% !important;
  height: auto !important;
}

.player {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 1% 10%;
}

.corner {
  width: 8%;
}

.middle {
  width: 90%;
  display: flex;
  justify-content: center;
  align-items: center;
}

.command {
  width: 90%;
}

/* Tooltip container */
.tooltip {
  position: relative;
  display: inline-block;
  /* If you want dots under the hoverable text */
}

/* Tooltip text */
.tooltip .tooltiptext {
  visibility: hidden;
  background-color: #8e93a6;
  color: #fff;
  width: 260px;
  text-align: left;
  padding: 5px;
  border-radius: 6px;

  /* Position the tooltip text */
  position: absolute;
  z-index: 1;
  bottom: 125%;
  left: 50%;
  margin-left: -130px;

  /* Fade in tooltip */
  opacity: 0;
  transition: opacity 0.3s;
}

/* Tooltip arrow */
.tooltip .tooltiptext::after {
  content: "";
  position: absolute;
  top: 100%;
  left: 50%;
  margin-left: -5px;
  border-width: 5px;
  border-style: solid;
  border-color: #8e93a6 transparent transparent transparent;
}

/* Show the tooltip text when you mouse over the tooltip container */
.tooltip:hover .tooltiptext {
  visibility: visible;
  opacity: 1;
}
</style>
