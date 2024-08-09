<template>
  <div id="id" class="container">
    <div v-if="state.matchId == null" class="createMatch">
      <div class="button-container">
        <button @click="matchRequest.countX = 4; matchRequest.countY = 4;" class="easy-button">Leicht</button>
        <button @click="matchRequest.countX = 8; matchRequest.countY = 8;" class="medium-button">Mittel</button>
        <button @click="matchRequest.countX = 16; matchRequest.countY = 16;" class="hard-button">Schwer</button>
      </div>
      <Configuration />
      <form @submit.prevent="createMatch">
        <p><label>
            Count X:
            <input type="number" v-model="matchRequest.countX" required />
          </label></p>
        <p><label>
            Count Y:
            <input type="number" v-model="matchRequest.countY" required />
          </label></p>
        <p><label>
            Player 1:
            <input type="string" v-model="matchRequest.player1" required />
          </label></p>
        <p><label>
            Player 2:
            <input type="string" v-model="matchRequest.player2" required />
          </label></p>
        <p style="margin-top: 50px;"><button class="start-button" type="submit">Spiel starten</button></p>
      </form>


    </div>
    <div v-if="state.matchId != null" class="match">
      <p class="text text-0">Current Player: {{ state.currentPlayer }}</p>
      <p class="text text-1">{{ matchRequest.player1 }}: {{ state.points1 }} Punkte</p>
      <p class="text text-2">{{ matchRequest.player2 }}: {{ state.points2 }} Punkte</p>
      <div class="field" :style="fieldStyle">
        <div v-for="(square, index) in state.squares" :key="index"
          :class="[square.closed ? square.winner == matchRequest.player1 ? 'square-closed-1' : 'square-closed-2' : 'square-open']"
          :style="squareStyle">
          <div v-for="(wall, id) in square.walls" :key="id">
            <button @click.prevent="setWall(wall.id, !wall.closed)" :disabled="wall.closed" :class="[wall.closed ? 'wall-closed' : 'wall-open',
    wall == square.leftWall ? 'left-wall' : ' ',
    wall == square.upperWall ? 'upper-wall' : ' ',
    wall == square.rightWall ? 'right-wall' : ' ',
    wall == square.bottomWall ? 'bottom-wall' : ' ']"></button>
          </div>
        </div>
      </div>
      <button @click="reset()" class="reset-button">Zurücksetzen</button>
      <button @click="createMatch()" class="restart-button">Neu
        starten</button>
    </div>
  </div>
</template>

<script setup>
//TODO:Komplette Spielfeld aufzubauen
//Field Breite und Höhe anpassen
//Doppelwände
//Players
//anpassen isclosed in squares
import axios from 'axios';
import { computed, reactive, watch } from 'vue';
import Configuration from './Configuration.vue';


const matchRequest = reactive({
  player1: '',
  player2: '',
  countX: 0,
  countY: 0,
});
const wallRequest = {
  id: '',
  closed: false,
}

const state = reactive({
  matchId: null,
  squares: [],
  widthX: null,
  heightY: null,
  currentPlayer: null,
  rep: null,
  points1: null,
  points2: null
})

const squareStyle = computed(() => {
  const width = 100 / state.widthX;
  const height = 100 / state.heightY;
  return {
    width: `${width}%`,
    height: `${height}%`,
    boxSizing: 'border-box'
  };
})
const fieldStyle = computed(() => {
  const proportion = state.heightY / state.widthX;
  const height = 400 * proportion;
  return {
    height: `${height}px`
  }
})


async function createMatch() {
  //if (state.matchId == null) { //???
  try {
    const response = await axios.post('http://localhost:8080/', matchRequest);
    state.matchId = response.data; // ID
    showMatch(state.matchId);
    console.log('MatchID ', state.matchId);
    state.widthX = matchRequest.countX;
    state.heightY = matchRequest.countY;
    state.currentPlayer = matchRequest.player1;
    state.points1 = 0;
    state.points2 = 0;

  } catch (error) {
    console.error('There was an error creating the match!', error);
  }
  //}
}
async function showMatch(matchId) {
  try {
    const response = await axios.get('http://localhost:8080/matches/' + matchId + '/field/squares');
    console.log('response ', response)
    state.squares = response.data;
    state.squares.forEach(square => {
      square.walls = [square.upperWall, square.leftWall, square.rightWall, square.bottomWall];
      console.log('showsquarewalls ', square.walls);
      console.log('isclosed ', isClosed(square))
    })
    console.log('showsquares ', state.squares);
  } catch (error) {
    console.error('There was an error showing the match!', error);
  }
}
async function setWall(id, closed) {
  console.log('set ', state.squares)
  try {
    axios.put('http://localhost:8080/walls/' + id, { id, closed }).then((response) => {
      console.log('wall was set ', response)
      state.rep = false;
      if (response.data) {
        state.squares.forEach(square => {
          square.walls.forEach(wall => {
            if (wall.id == response.data.id) {
              wall.closed = response.data.closed
              if (isClosed(square)) {
                square.closed = true;
                state.rep = true;
                square.winner = state.currentPlayer;
                pointsToPlayer();
              }
            }
          })
        });
      }
      //getNeighbours(state.matchId, id)
      console.log('squares ', state.squares)
      const playersArray = [matchRequest.player1, matchRequest.player2]
      for (let player of playersArray) {
        if (player != state.currentPlayer && !state.rep) {
          state.currentPlayer = player
          break
        }
      }
    })
  } catch (error) {
    console.error('There was an error changing the wall!', error);
  }
}
async function getNeighbours(matchId, id) {
  try {
    const response = await axios.get('http://localhost:8080/matches/' + matchId + '/field/walls/' + id + '/neighbours');
    console.log('neighbours: ', response);
    response.data.forEach(square => {

      console.log('neigboursclosed ', square.closed);
    })
  } catch (error) {
    console.error('There was an error showing neighbours!', error);
  }
}
function isClosed(square) {
  return square.leftWall.closed && square.upperWall.closed && square.rightWall.closed && square.bottomWall.closed;
}
function pointsToPlayer() {
  if (state.currentPlayer == matchRequest.player1) {
    state.points1 += 1;
  } else if (state.currentPlayer == matchRequest.player2) {
    state.points2 += 1;
  }
}
function reset() {
  state.matchId = null;
  matchRequest.player1 = '';
  matchRequest.player2 = '';
  matchRequest.countX = 0;
  matchRequest.countY = 0;
}
watch(() => state.squares, (newVal) => {
  console.log('watch: ', newVal)
}, {
  immediate: true
})
</script>
<style scoped>
.container {
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>