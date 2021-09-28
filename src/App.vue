<template lang="pug">
.container
  .header
    span(v-for="voter in voters") {{ voter }}
  row
    pick(v-for="firstChoice in firstChoices", :top="winners.has(firstChoice.pick)", :bottom="firstLosers.includes(firstChoice.pick)", :style="`color: ${getColor(firstChoice.pick)}`") {{ firstChoice.pick }}
  row
    pick(v-for="secondChoice in secondChoices", :bottom="firstLosers.includes(secondChoice.pick)", :style="`color: ${getColor(secondChoice.pick)}`") {{ secondChoice.pick }}
  row
    pick(v-for="thirdChoice in thirdChoices", :bottom="firstLosers.includes(thirdChoice.pick)", :style="`color: ${getColor(thirdChoice.pick)}`") {{ thirdChoice.pick }}
  button(@click="onStartTally") Start Tally
  hr(style="width: 100%")
  span(style="text-align: center; font-size: 20px;") {{ narrationText }}
  span(style="text-align: center; font-size: 20px") 
    strong Score: {{ score }}
</template>

<script lang="ts">
import axios from "axios";
import styled from "vue3-styled-components";
import { defineComponent } from "vue";
import { Obj } from "@/types";

const Row = styled.div`
  display: grid;
  grid-template-rows: 1fr;
  grid-template-columns: repeat(9, 1fr);
  grid-gap: 10px;
`;
const Pick = styled("span", { top: Boolean, bottom: Boolean })`
  border: solid thin black;
  display: grid;
  align-content: center;
  justify-content: center;
  border-radius: 10px;
  transition: all 1s ease-in-out;
  transform: ${props => (props.top ? "translateY(350px)" : "")};
  opacity: ${props => (props.bottom ? "0" : "")};
`;

type Data = {
  voters: string[];
  firstChoices: { pick: string; active: boolean }[];
  secondChoices: { pick: string; active: boolean }[];
  thirdChoices: { pick: string; active: boolean }[];
  winners: Set<string>;
  firstLosers: string[];
  counting: boolean;
  narrationText: string;
  score: string;
  allBooks: string[];
  color: Obj;
};
type PickType = {
  pick: string;
  active: boolean;
};

export default defineComponent({
  name: "App",
  data: (): Data => ({
    voters: [],
    firstChoices: [],
    secondChoices: [],
    thirdChoices: [],
    winners: new Set([]),
    firstLosers: [],
    counting: false,
    narrationText: ``,
    score: ``,
    allBooks: [],
    color: {}
  }),
  components: {
    row: Row,
    pick: Pick
  },
  async mounted() {
    try {
      const { data }: { data: Obj[] } = await axios.get(
        `http://localhost:3000/votes`,
        {
          headers: {
            "Content-Type": "application/json"
          }
        }
      );
      const bookNames: string[] = data.reduce(
        (arr: string[], obj: Obj) => [...arr, ...Object.values(obj)],
        []
      );
      this.voters = Object.keys(data[0]);
      this.firstChoices = Object.values(data[0]).map(createPickObj);
      this.secondChoices = Object.values(data[1]).map(createPickObj);
      this.thirdChoices = Object.values(data[2]).map(createPickObj);
      // set all books to winners initially
      this.allBooks = bookNames;
      console.log(`DATA`, data);
      console.log(`books`, this.winners);
    } catch (error) {
      console.log(`ERROR`, error);
    }
  },
  methods: {
    onStartTally() {
      this.narrationText = `Advancing top picks`;
      this.counting = true;
      const count: Obj = this.firstChoices.reduce(
        (tally: Obj, pick: PickType) => {
          if (!tally[pick.pick]) return { ...tally, [pick.pick]: 1 };
          return { ...tally, [pick.pick]: tally[pick.pick] + 1 };
        },
        {}
      );
      const newWinners = Object.keys(count).filter(
        (book: string) => count[book] > 1
      );
      this.winners = new Set(newWinners);
      setTimeout(() => {
        this.narrationText = `Removing losers`;
        this.firstLosers = this.allBooks.filter(
          (book: string) => count[book] <= 1 || !count[book]
        );
      }, 2000);
      setTimeout(() => {
        this.score = Object.keys(count).reduce((score: string, key: string) => {
          if (count[key] > 1) {
            if (!score) return `${key}: ${count[key]}`;
            return `${score}, ${key}: ${count[key]}`;
          }
          return score;
        }, ``);
        this.narrationText = `Removing losers`;
        this.firstLosers = this.allBooks.filter(
          (book: string) => count[book] <= 1 || !count[book]
        );
      }, 2000);
    },
    getColor(book: string): string {
      if (!this.color[book]) {
        const color = getRandomColor();
        this.color = { ...this.color, [book]: color };
        return color;
      }
      return this.color[book];
    }
  }
});

function getRandomColor() {
  const chars = Array(6)
    .fill(``)
    .map(getCh);
  console.log(`CAHRS`, chars);
  return `#${chars.join(``)}`;
}
function getCh() {
  const chars = `0123456789ABCDEF`;
  const index = Math.floor(Math.random() * chars.length);
  return chars[index];
}
function createPickObj(pick: string): PickType {
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

p {
  background-color: blue;
  transition: all 1s ease-in-out;
}
p:hover {
  opacity: 0;
}
</style>
