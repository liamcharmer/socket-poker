<template>
  <div class="playing__deck">
    <template v-if="stats && deck !== null">
      <p>Number of cards in deck: {{ deck.length }}</p>
    </template>
    <div class="deck">
      <Card
        class="flipped"
        v-for="(card, index) in deck"
        v-bind:key="index"
        :card="card"
      />
    </div>
  </div>
</template>

<script>
export default {
  props: {
    deck: {
      type: Array,
      default: [],
      required: true
    },
    stats: {
      type: Boolean,
      default: false
    }
  }
};
</script>

<style></style>
