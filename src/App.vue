<template lang="pug">
.container
  .header
    span(v-for="voter in voters") {{ voter }}
  row
    pick(v-for="firstChoice in firstChoices", :top="winners.has(firstChoice.pick)", :bottom="firstLosers.includes(firstChoice.pick)", :style="`background-color: ${getColor(firstChoice.pick)}`") {{ firstChoice.pick }}
  row
    pick(
      v-for="secondChoice in secondChoices", 
      :top="isSecondRoundWinner(secondChoice)", 
      :sec-top="hasVotedRecord[secondChoice.voter] === false", 
      :bottom="firstLosers.includes(secondChoice.pick)", 
      :style="`background-color: ${getColor(secondChoice.pick)}`"
      ) {{ secondChoice.pick }}
  row
    pick(v-for="thirdChoice in thirdChoices", :bottom="firstLosers.includes(thirdChoice.pick)", :style="`background-color: ${getColor(thirdChoice.pick)}`") {{ thirdChoice.pick }}
  button(@click="onStartTally") Start Tally
  hr(style="width: 100%")
  span(style="text-align: center; font-size: 20px;") {{ narrationText }}
  span(style="text-align: center; font-size: 20px") 
    strong Score: {{ score }}
  h1(style="padding-top: 150px; text-align: center; margin: 0;") {{ finalText }} 
  h2(style="padding-top: 0px; text-align: center; margin: 0;") {{ secText }} 
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
const Pick = styled("span", { top: Boolean, bottom: Boolean, secTop: Boolean })`
  border: solid thin black;
  display: grid;
  align-content: center;
  justify-content: center;
  border-radius: 10px;
  transition: all 1s ease-in-out;
  transform: ${props => (props.top ? "translateY(350px)" : "")};
  transform: ${props => (props.secTop ? "translateY(280px)" : "")};
  opacity: ${props => (props.bottom ? "0" : "")};
`;

type PickType = {
  pick: string;
  voter: string;
};
type Data = {
  voters: string[];
  firstChoices: PickType[];
  secondChoices: PickType[];
  thirdChoices: PickType[];
  winners: Set<string>;
  firstLosers: string[];
  counting: boolean;
  narrationText: string;
  score: string;
  allBooks: string[];
  color: Obj;
  foo: boolean;
  data: Obj[];
  hasVotedRecord: Obj;
  count: Obj;
  finalText: string;
  secText: string;
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
    color: {},
    foo: false,
    data: [],
    hasVotedRecord: {},
    count: {},
    finalText: ``,
    secText: ``,
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
      this.data = data;
      this.voters = Object.keys(data[0]);
      this.firstChoices = Object.keys(data[0]).map(voter =>
        createPickObj(data[0], voter)
      );
      this.secondChoices = Object.keys(data[1]).map(voter =>
        createPickObj(data[1], voter)
      );
      this.thirdChoices = Object.keys(data[2]).map(voter =>
        createPickObj(data[2], voter)
      );
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
      this.count = count;
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
        // make lowest picks disappear
        this.firstLosers = this.allBooks.filter(
          (book: string) => count[book] <= 1 || !count[book]
        );
        // remove loser picsk from count
        this.count = this.removeLosersFromCount(1);
      }, 4000);
      setTimeout(() => {
        this.narrationText = `Tallying 2nd round votes`;
        const firstRound = this.data[0];
        const hasVoted = (voter: string) =>
          !this.firstLosers.includes(firstRound[voter]);
        const hasVotedRecord = Object.keys(firstRound).reduce(
          (acc: Obj, voter: string) => {
            if (hasVoted(voter)) return { ...acc, [voter]: true };
            return { ...acc, [voter]: false };
          },
          {}
        );
        this.hasVotedRecord = hasVotedRecord;
        // add second round votes
        console.log(`NEW COUNT`, this.getNewCount(2));
        this.count = this.getNewCount(2);
        // update hasVotedRecord
      }, 5000);
      setTimeout(() => {
        // make lowest disappear
        this.narrationText = `Removing picks with lowest votes`;
        this.firstLosers = [...this.firstLosers, ...this.getNewLosers()];
        // update hasVotedRecord so losers can use next vote
        // TODO: map over hasVotedRecord and if a voter voted for a loser switch their value to false (use this.data[1])
        console.log(`losers`, this.firstLosers)
        // const newRecord = Object.keys(this.hasVotedRecord).reduce((acc, voter) => {
        //   if (this.firstLosers.includes(this.data[1][voter]) && this.firstLosers.includes(this.data[0][voter])) return { ...acc, [voter]: false }
        //   return { ...acc, [voter]: true }
        // }, {})
      }, 8000);
      setTimeout(() => {
        this.hasVotedRecord = { ...this.hasVotedRecord, Shay: false }
        this.score = `Dune: 3, Murderbot Diaries (#1): 4`
        // update count
        this.count = this.removeLosersFromCount(2);
        // update score
        this.score = this.getNewScore();
      }, 9000)
      setTimeout(() => {
        console.log(`firstlosers`, this.firstLosers)
        console.log(`COUNT`, this.count)
        console.log(`SCORE`, this.score)
        console.log(`RECORD`, this.hasVotedRecord)
        // this.firstLosers = [...this.firstLosers, `Dune`]
        this.finalText = `Murderbot Diaries (#1) Wins!`
        this.secText = `Edges out Dune 4 to 3`
      }, 10000);
    },
    getColor(book: string): string {
      if (!this.color[book]) {
        const color = getRandomColor();
        this.color = { ...this.color, [book]: color };
        return color;
      }
      return this.color[book];
    },
    isSecondRoundWinner(pick: PickType) {
      console.log(`PICK`, pick);
      // console.log(`PICK`, pick);
      // return true.foo;
    },
    // also updates hasVotedRecord
    getNewCount(round: number) {
      const votes = this.data[round - 1];
      return Object.keys(this.hasVotedRecord).reduce(
        (acc: Obj, voter: string) => {
          const pick = votes[voter];
          if (this.hasVotedRecord[voter] === false) {
            // TODO: must do this after updating hasVotedRecord
            // this.hasVotedRecord = { ...this.hasVotedRecord, [voter]: true };
            return { ...acc, [pick]: acc[pick] + 1 };
          }
          return acc;
        },
        this.count
      );
    },
    getNewScore() {
      return Object.keys(this.count).reduce((score: string, key: string) => {
        if (this.count[key] > 1) {
          if (!score) return `${key}: ${this.count[key]}`;
          return `${score}, ${key}: ${this.count[key]}`;
        }
        return score;
      }, ``);
    },
    getNewLosers() {
      // TODO: handle ties
      return Object.keys(this.count).reduce((acc: string[], book: string) => {
        if (this.count[book] < 3) return [...acc, book];
        return acc;
      }, []);
    },
    removeLosersFromCount(min: number) {
      return Object.keys(this.count).reduce((acc, book) => {
        if (this.count[book] > min) return { ...acc, [book]: this.count[book] };
        return acc;
      }, {});
    }
  }
});

function getRandomColor() {
  const chars = Array(6)
    .fill(``)
    .map(getCh);
  return `#${chars.join(``)}`;
}
function getCh() {
  const chars = `0123456789ABCDEF`;
  const index = Math.floor(Math.random() * chars.length);
  return chars[index];
}
function createPickObj(votes: Obj, voter: string): PickType {
  return { pick: votes[voter], voter };
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
</style>
