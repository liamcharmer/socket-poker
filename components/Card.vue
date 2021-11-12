<template>
  <div class="card" v-bind:class="`${card.suit} value--${card.value}`">
    <div class="front">
      <div class="top__left">
        <h3 class="value">{{ card.value }}</h3>
        <div
          class="icon"
          v-html="require(`../assets/icons/${card.suit}.svg?raw`)"
        />
      </div>
      <div class="bottom__right">
        <h3 class="value">{{ card.value }}</h3>
        <div
          class="icon"
          v-html="require(`../assets/icons/${card.suit}.svg?raw`)"
        />
      </div>
      <div class="diagram">
        <template v-if="parseInt(card.value)">
          <div
            class="icon"
            v-for="iconIndex in parseInt(card.value)"
            v-bind:key="iconIndex"
            v-html="require(`../assets/icons/${card.suit}.svg?raw`)"
          />
        </template>
        <template v-else-if="card.value == 'A'">
          <div
            class="icon"
            v-html="require(`../assets/icons/${card.suit}.svg?raw`)"
          />
        </template>
        <template v-else-if="card.value == 'K'">
          <img src="~/assets/images/naruto/king.jpeg" alt="" />
        </template>
        <template v-else-if="card.value == 'Q'">
          <img src="~/assets/images/naruto/queen.jpeg" alt="" />
        </template>
        <template v-else-if="card.value == 'J'">
          <img src="~/assets/images/naruto/jack.jpeg" alt="" />
        </template>
      </div>
    </div>
    <div class="back">
      <div class="pattern"></div>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    card: {
      type: Object,
      default: null,
      required: true
    }
  }
};
</script>

<style></style>
