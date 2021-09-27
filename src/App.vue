<template lang="pug">
.container
  .header
    span(v-for="voter in voters") {{ voter }}
  row
    pick(v-for="firstChoice in firstChoices", :top="frontRunners.includes(firstChoice.pick)") {{ firstChoice.pick }}
  row
    span(v-for="secondChoice in secondChoices") {{ secondChoice.pick }}
  row
    span(v-for="thirdChoice in thirdChoices") {{ thirdChoice.pick }}
  button(@click="onStartTally") Start Tally
  hr(style="width: 100%")
</template>

<script lang="ts">
import axios from "axios";
import styled from "vue3-styled-components";
import { defineComponent } from "vue";

const Row = styled.div`
  display: grid;
  grid-template-rows: 1fr;
  grid-template-columns: repeat(9, 1fr);
  grid-gap: 10px;
`;
const Pick = styled("span", { top: Boolean })`
  background-color: gray;
  transition: all 1s ease-in-out;
  transform: ${props => (props.top ? "translateY(200px)" : "")};
`;

type Data = {
  voters: string[];
  firstChoices: { pick: string; active: boolean }[];
  secondChoices: { pick: string; active: boolean }[];
  thirdChoices: { pick: string; active: boolean }[];
  frontRunners: string[];
  counting: boolean;
};

export default defineComponent({
  name: "App",
  data: (): Data => ({
    voters: [],
    firstChoices: [],
    secondChoices: [],
    thirdChoices: [],
    frontRunners: [],
    counting: false
  }),
  components: {
    row: Row,
    pick: Pick
  },
  async mounted() {
    try {
      const { data }: { data: object[] } = await axios.get(
        `http://localhost:3000/votes`,
        {
          headers: {
            "Content-Type": "application/json"
          }
        }
      );
      this.voters = Object.keys(data[0]);
      this.firstChoices = Object.values(data[0]).map(createPickObj);
      this.secondChoices = Object.values(data[1]).map(createPickObj);
      this.thirdChoices = Object.values(data[2]).map(createPickObj);
      console.log(`DATA`, data);
    } catch (error) {
      console.log(`ERROR`, error);
    }
  },
  methods: {
    onStartTally() {
      this.counting = true;
      this.frontRunners = ["It"];
    },
    isTopTeir() {}
  }
});

function createPickObj(pick: string): { pick: string; active: boolean } {
  return { pick, active: true };
}
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
.fade-enter-active,
.fade-leave-active {
  transition: opacity 1s;
}
.fade-enter, .fade-leave-to /* .fade-leave-active below version 2.1.8 */ {
  opacity: 0;
}
</style>
