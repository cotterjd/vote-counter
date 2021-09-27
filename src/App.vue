<template lang="pug">
.container
  .header
    span(v-for="voter in voters") {{ voter }}
  row
    span(v-for="firstChoice in firstChoices") {{ firstChoice.pick }}
  row
    span(v-for="secondChoice in secondChoices") {{ secondChoice }}
  row
    span(v-for="thirdChoice in thirdChoices") {{ thirdChoice }}
  span(v-for="frontrunner in frontRunners") {{ frontrunner }}
  button(@click="onStartTally") Start Tally
  hr(style="width: 100%")
</template>

<script lang="ts">
import axios from "axios";
import styled from "vue3-styled-components";

const Row = styled.div`
  display: grid;
  grid-template-rows: 1fr;
  grid-template-columns: repeat(9, 1fr);
  grid-gap: 10px;

  span {
    background-color: gray;
  }
`;

export default {
  name: "App",
  data: () => ({
    voters: [],
    firstChoices: [],
    secondChoices: [],
    thirdChoices: [],
    frontRunners: [],
    counting: false
  }),
  components: {
    row: Row
  },
  async mounted() {
    try {
      const { data } = await axios.get(`http://localhost:3000/votes`, {
        headers: {
          "Content-Type": "application/json"
        }
      });
      this.voters = Object.keys(data[0]);
      this.firstChoices = Object.values(data[0]).map(pick => ({
        pick,
        active: true
      }));
      this.secondChoices = Object.values(data[1]);
      this.thirdChoices = Object.values(data[2]);
      console.log(`DATA`, data);
    } catch (error) {
      console.log(`ERROR`, error);
    }
  },
  onStartTally() {
    this.counting = true;
  }
};
</script>

<style scoped>
.container {
  display: grid;
  grid-template-columns: 1fr;
  grid-template-rows: repeat(4, 1fr);
  grid-gap: 10px;
}
.header {
  display: grid;
  grid-template-rows: 1fr;
  grid-template-columns: repeat(9, 1fr);
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
