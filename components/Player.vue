<template>
  <div class="player">
    <h2>{{ player.name }}</h2>
    <p>{{ player }}</p>
    <p>Player Chips: {{ player.chips }}</p>
    <p>Player Pot {{ player.pot }}</p>
    <p>Player last bet: {{ player.last_bet }}</p>
    <p>Player Round Pot: {{ player.round_pot }}</p>
    <p v-if="player.dealer">Dealer</p>
    <p v-if="player.small_blind">Small Blind</p>
    <p v-if="player.big_blind">Big Blind</p>
    <p v-if="player.betting_active">Currently Active</p>
    <p>Hand Rank: {{ handScore }}</p>
    <template v-if="player.betting_active">
      <button @click="fold()">Fold</button>
      <button @check="check()" v-if="table.round_pot == 0">Check</button>
      <button @click="call()" v-if="table.round_pot !== 0">Call</button>
      <button @click="raising = !raising">
        <span v-if="table.round_pot !== 0">Raise</span><span v-else>Bet</span>
      </button>
      <form @submit.prevent="raise()">
        <input
          type="number"
          :min="table.minimum_bet + table.raise_difference"
          v-if="raising && table.round_pot !== 0"
          v-model="raiseAmount"
        />
        <input
          type="number"
          :min="table.minimum_bet"
          v-else-if="raising"
          v-model="raiseAmount"
        />
      </form>
    </template>
    <hr />
    <template v-if="typeof player.cards === 'string'"> </template>
    <template v-else>
      <Card
        :card="card"
        v-for="(card, index) in player.cards"
        v-bind:key="index"
      />
    </template>
    <div class="actions">
      <button @click="reviewHand()">Review Hand</button>
    </div>
  </div>
</template>

<script>
const Hand = require("pokersolver").Hand;
export default {
  watch: {
    "table.community_cards"(old, newTable) {
      this.playerHandScore();
    }
  },
  props: {
    playerIndex: {
      type: Number,
      required: true
    },

    player: {
      type: Object,
      required: true
    },
    table: {
      type: Object,
      required: false
    },
    roundNumber: {
      type: Number,
      required: true
    },
    gameStatus: {
      type: String,
      required: true
    }
  },
  methods: {
    check() {
      console.log("Checking");
      this.$emit("check", this.playerIndex);
    },
    call() {
      console.log("Calling");
      this.$emit("call", this.playerIndex);
    },
    raise() {
      console.log("Raising");
      console.log(this.raiseAmount);
      this.$emit("raise", {
        playerIndex: this.playerIndex,
        value: this.raiseAmount
      });
    },
    fold() {
      this.$emit("fold", this.playerIndex);
    },
    reviewHand() {
      this.$emit("reviewHand", this.playerIndex);
    },
    playerHandScore() {
      try {
        let playerCards = this.player.cards;
        if (typeof playerCards == "string") {
          playerCards = JSON.parse(this.player.cards);

          if (typeof playerCards == "string") {
            playerCards = [];
          }
        }
        if (playerCards.length >= 2) {
          let allCards = this.table.community_cards.concat(playerCards);
          let solverFormat = [];
          for (let c = 0; c < allCards.length; c++) {
            if (allCards[c] && typeof allCards[c] !== "string") {
              let value = allCards[c].value;
              if (value == 10) {
                value = "T";
              }
              let cardString = value + allCards[c].suit[0];
              solverFormat.push(cardString);
            }
          }
          if (solverFormat.length > 0) {
            let hand = Hand.solve(solverFormat);
            this.handScore = hand.rank;
          }
        }
      } catch (e) {
        console.log("Reviewing hand Error");
        console.log(e);
      }
    }
  },
  data() {
    return {
      chips: 0,
      handScore: 0,
      raising: false,
      raiseAmount: 0
    };
  }
};
</script>

<style></style>
