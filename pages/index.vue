<template>
  <main>
    <h2>Rooms</h2>
    <template v-if="selectedRoom == null">
      <div v-for="table in tables" v-bind:key="table.id">
        {{ table }}
      </div>
      <button @click="getTables()">Get Tables</button>
      <h2>Create Table</h2>
      <form @submit.prevent="createTable(0)">
        <p>Max number of Players</p>
        <input type="number" v-model="createTableForm.maxPlayers" />
        <p>Starting Number of Chips</p>
        <input
          type="number"
          placeholder="Starting Number of Chips"
          v-model="createTableForm.startingChips"
        />
        <p>Big Blind Amount</p>
        <input
          type="number"
          placeholder="Big Blind Amount"
          v-model="createTableForm.bigBlind"
        />
        <p>Small Blind Amount = {{ createTableForm.bigBlind / 2 }}</p>
        <p>
          Blind is {{ createTableForm.bigBlind / 2 }} /
          {{ createTableForm.bigBlind }}
        </p>
        <p>Blind Increase over time Mode</p>
        <input type="checkbox" v-model="createTableForm.blindIncreaseMode" />
        <template v-if="createTableForm.blindIncreaseMode">
          <p>Blind Increase Time (Milliseconds)</p>
          <input
            type="number"
            placeholder="Blind Increase Time (Milliseconds)"
            v-model="createTableForm.blindIncreaseTime"
          />
        </template>

        <p>Real Money Mode</p>
        <input type="checkbox" v-model="createTableForm.realMoneyMode" />
        <template v-if="createTableForm.realMoneyMode">
          <p>Minimum real money entry</p>
          <input
            type="number"
            placeholder="Minimum Real Money Entry"
            v-model="createTableForm.realMoneyMinEntry"
          />
          <p>£1 / $1 = How many Chips?</p>
          <input type="number" v-model="createTableForm.realMoneyChipEqual" />
        </template>
        <p>Your Name</p>
        <input
          type="text"
          v-model="createPlayerForm.playerName"
          placeholder="Your Name"
        />
        <button type="submit">Create Room</button>
      </form>
      <hr />
      <h2>Join Table</h2>
      <form action="" @submit.prevent="joinRoom">
        <input
          type="text"
          placeholder="Room Code"
          v-model="joinRoomForm.roomCode"
        />
      </form>
    </template>
    <template v-if="selectedRoom">
      <h2>Room</h2>
      <p>
        Room Code: {{ selectedRoom.join_code }}, Starting Chips:
        {{ selectedRoom.starting_chips }}, Blind:
        {{ selectedRoom.small_blind }} / {{ selectedRoom.big_blind }}, Max
        Players: {{ selectedRoom.max_players }}, Round Pot
        {{ selectedRoom.round_pot }}, Entire Pot: {{ selectedRoom.entire_pot }},
        Minimum bet: {{ selectedRoom.minimum_bet }}, Latest Raise:
        {{ selectedRoom.latest_raise }}, Raise Difference:
        {{ selectedRoom.raise_difference }}<br />
        <template v-if="selectedRoom.game_started == false">
          <button @click="startGame()">
            Start Game
          </button>
        </template>
        <template v-else>
          <button @click="shuffleDeck(true)">Shuffle Deck</button>
          <button @click="startRound()">Start Round</button>
          <button @click="revealCards(3, 'flop')">Start Flop</button>
          <button @click="revealCards(1, 'turn')">Start Turn</button>
          <button @click="revealCards(1, 'river')">Start River</button>
          <button @click="startBettingRound()">Start Betting Round</button>
          <button @click="endRound()">End Round</button>
        </template>
      </p>
      <form
        action=""
        @submit.prevent="playerJoinRoom(false)"
        v-if="devicePlayer == null"
      >
        <h2>Join the Room</h2>
        <input
          type="text"
          placeholder="Player Name"
          v-model="createPlayerForm.playerName"
        />
      </form>
      <h2>Players</h2>
      <p>Dealer Number: {{ dealerNumber }}</p>
      <div class="players">
        <Player
          v-for="(player, index) in playerList"
          v-bind:key="player.id"
          :player="player"
          :table="selectedRoom"
          :playerIndex="index"
          :dealerNumber="dealerNumber"
          :roundNumber="roundNumber"
          :gameStatus="gameStatus"
          @fold="playerFolded"
          @reviewHand="playerReviewHand"
          @call="playerCalled"
          @raise="playerRaised"
          @check="playerChecked"
          v-bind:class="{
            'device--player':
              devicePlayer != null && player.id == devicePlayer.id,
            'currently--active': player.betting_active
          }"
        />
      </div>
      <div class="decks">
        <div class="duo">
          <div>
            <h2>Folded Deck</h2>
            <Deck :deck="selectedRoom.folded_deck" :stats="true" />
          </div>
          <div>
            <h2>Deck</h2>
            <Deck :deck="selectedRoom.deck" :stats="true" />
          </div>
          <div>
            <h2>Community Cards - {{ selectedRoom.game_status }}</h2>
            <div class="duo">
              <Card
                :card="card"
                v-for="(card, index) in selectedRoom.community_cards"
                v-bind:key="index"
              />
            </div>
          </div>
        </div>
      </div>
    </template>

    <!-- <h2>Card View</h2>
    <div class="cards">
      <div class="duo" v-for="(card, index) in deck" v-bind:key="index">
        <Card :card="card" />
        <Card class="flipped" :card="card" />
      </div>
    </div> -->
  </main>
</template>

<script>
export default {
  computed: {
    playerList() {
      return this.players;
    }
  },
  data() {
    return {
      suits: ["spades", "diamonds", "clubs", "hearts"],
      values: [
        "A",
        "2",
        "3",
        "4",
        "5",
        "6",
        "7",
        "8",
        "9",
        "10",
        "J",
        "Q",
        "K"
      ],
      players: [],
      firstCardDeal: true,
      dealerNumber: 0,
      smallBlindNumber: 0,
      bigBlindNumber: 0,
      roundNumber: 0,
      gameStatus: "preflop",
      tables: [],
      tableRealTime: null,
      playerRealTime: null,
      joinRoomForm: {
        roomCode: ""
      },
      selectedRoom: null,
      createPlayerForm: {
        playerName: ""
      },
      createTableForm: {
        startingChips: 0,
        smallBlind: 0,
        bigBlind: 0,
        blindIncreaseMode: false,
        blindIncreaseTime: 0,
        realMoneyMode: false,
        realMoneyMinEntry: 0,
        realMoneyChipEqual: 0,
        maxPlayers: 8,
        minimumBet: 0
      },
      devicePlayer: null
    };
  },
  methods: {
    async playerJoinRoom(host = false) {
      let playerData = {
        name: this.createPlayerForm.playerName,
        room_id: this.selectedRoom.id,
        chips: this.selectedRoom.starting_chips,
        host: host,
        cards: []
      };
      console.log(playerData);
      const { data, error } = await this.$supabase
        .from("players")
        .insert([playerData]);
      if (error) {
        console.log(error);
      } else {
        this.devicePlayer = data[0];
      }
    },
    async joinRoom() {
      let tables = await this.$supabase
        .from("poker rooms")
        .select("*")
        .eq("join_code", this.joinRoomForm.roomCode);
      if (tables.error) {
        console.log(tables.error);
      } else {
        if (tables.data.length > 0) {
          this.selectedRoom = tables.data[0];
          this.setupTableUpdates();
          let players = await this.$supabase
            .from("players")
            .select("*")
            .eq("room_id", this.selectedRoom.id);
          if (players.error) {
            console.log(players.error);
          } else {
            let playersJSON = players.data.map(player => {
              return player;
            });
            this.players = playersJSON;
            this.setupTablePlayerUpdates();
          }
        } else {
          alert("No Room Exists");
        }
      }
    },
    async setupTablePlayerUpdates() {
      console.log(this.selectedRoom.id);
      this.playerRealTime = this.$supabase
        .from(`players:room_id=eq.${this.selectedRoom.id}`)
        .on("INSERT", this.playerInserted)
        .on("UPDATE", this.playerUpdated)
        .on("DELETE", this.playerDeleted)
        .subscribe();
    },
    async playerInserted(payload) {
      console.log("----PLAYER INSERTED----");
      let newPlayer = payload.new;
      newPlayer.cards = [];
      this.players.push(newPlayer);
    },

    async playerUpdated(payload) {
      console.log("----PLAYER UPDATED----");
      console.log(payload);
      let updatedPlayerIndex = this.players.findIndex(player => {
        if (player.id == payload.new.id) {
          return player;
        }
      });
      let newPlayer = payload.new;

      // newPlayer.cards = JSON.parse(newPlayer.cards);
      if (this.devicePlayer !== null && newPlayer.id == this.devicePlayer.id) {
        this.devicePlayer = newPlayer;
      }
      this.$set(this.players, updatedPlayerIndex, newPlayer);
      console.log(this.players[updatedPlayerIndex]);
      console.log("______");
    },

    async playerDeleted(payload) {
      console.log("----PLAYER DELETED----");
      console.log(payload);
    },
    async setupTableUpdates() {
      this.tableRealTime = this.$supabase
        .from(`poker rooms:id=eq.${this.selectedRoom.id}`)
        .on("UPDATE", this.tableUpdated)
        .on("DELETE", this.tableDeleted)
        .subscribe();
    },
    async tableInserted(payload) {
      console.log("----TABLE INSERTED----");
      this.tables.push(payload.new);
    },
    async tableUpdated(payload) {
      console.log("----TABLE UPDATED----");
      this.selectedRoom = payload.new;
    },
    async tableDeleted(payload) {
      console.log("----TABLE DELETED----");
      console.log(payload);
    },
    async updatePlayer(player) {
      console.log("----UPDATE PLAYER REQUEST----");
      let updatePlayer = player;
      console.log(updatePlayer);
      // updatePlayer.cards = JSON.stringify(updatePlayer.cards);
      console.log(player);
      const { data, error } = await this.$supabase
        .from("players")
        .update(updatePlayer)
        .match({ id: player.id });
    },
    async updateTable() {
      const { data, error } = await this.$supabase
        .from("poker rooms")
        .update(this.selectedRoom)
        .match({ id: this.selectedRoom.id });
    },
    async getTables() {
      let tables = await this.$supabase.from("poker rooms").select("*");
      this.tables = tables.data;
    },
    generateRoomCode() {
      var text = "";
      var possible = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";

      for (var i = 0; i < 5; i++)
        text += possible.charAt(Math.floor(Math.random() * possible.length));

      return text;
    },
    async createTable(attempt) {
      let generatedCode = this.generateRoomCode();
      let table = {
        join_code: generatedCode,
        starting_chips: this.createTableForm.startingChips,
        small_blind: this.createTableForm.bigBlind / 2,
        big_blind: this.createTableForm.bigBlind,
        blind_increase_mode: this.createTableForm.blindIncreaseMode,
        real_money_mode: this.createTableForm.realMoneyMode,
        max_players: this.createTableForm.maxPlayers
      };
      if (this.createTableForm.blindIncreaseMode) {
        table.blind_increase_time = this.createTableForm.blindIncreaseTime;
      }
      if (this.createTableForm.realMoneyMode) {
        table.real_money_min_entry = this.createTableForm.realMoneyMinEntry;
        table.real_money_chip_equal = this.createTableForm.realMoneyChipEqual;
      }

      const { data, error } = await this.$supabase
        .from("poker rooms")
        .insert([table]);

      if (error) {
        if (attempt >= 5) {
          alert("Error with server");
        } else {
          this.createTable(attempt + 1);
        }
      } else {
        console.log(data);
        this.joinRoomForm.roomCode = data[0].join_code;
        await this.joinRoom();
        this.playerJoinRoom(true);
      }
    },
    playerFolded(playerIndex) {
      this.removePlayerCards(playerIndex);
    },
    playerChecked(playerIndex) {},
    playerCalled(playerIndex) {
      console.log("player Called");
      let playersCurrentPot = this.players[playerIndex].pot;
      let amountToBet = this.selectedRoom.minimum_bet - playersCurrentPot;
      this.playerBet(amountToBet, playerIndex);
    },
    playerRaised(raise) {
      // raise bet going in is 500
      let playerIndex = raise.playerIndex;
      let value = raise.value;

      this.selectedRoom.minimum_bet = value;
      // latest_raise = 200
      this.selectedRoom.raise_difference =
        value - this.selectedRoom.latest_raise;

      if (this.selectedRoom.minimum_bet !== this.selectedRoom.big_blind) {
        this.selectedRoom.latest_raise = value;
      } else {
        this.selectedRoom.latest_raise;
      }
      // Latest raise = 500

      this.updateTable(true);
      this.playerBet(value, playerIndex);
    },
    playerReviewHand(playerIndex) {
      console.log(this.reviewHand(playerIndex));
    },
    reviewHand(playerIndex) {
      let allCards = this.selectedRoom.community_cards.concat(
        this.players[playerIndex].cards
      );
      let solverFormat = [];
      for (let c = 0; c < allCards.length; c++) {
        let value = allCards[c].value;
        if (value == 10) {
          value = "T";
        }
        let cardString = value + allCards[c].suit[0];
        solverFormat.push(cardString);
      }
      console.log(solverFormat);
      if (solverFormat.length > 0) {
        let hand = Hand.solve(solverFormat);

        return hand;
      }
    },
    addPlayer() {
      let player = {};
      player.startingChips = 200000;
      player.name = "Player Name";
      player.cards = [];
      this.players.push(player);
    },
    removePlayerCards(index) {
      if (typeof this.players[index].cards == "string") {
        console.log("Cards are a string");
        this.players[index].cards = JSON.parse(this.players[index].cards);

        if (typeof this.players[index].cards == "string") {
          this.players[index].cards = [];
        }
      }
      if (this.players[index].cards.length > 0) {
        console.log(
          "they have " + this.players[index].cards.length + " of cards "
        );
        let cardFolded = false;
        let numberOfCards = this.players[index].cards.length;
        for (let c = 0; c < numberOfCards; c++) {
          console.log("Loop " + c + " of " + this.players[index].cards.length);
          let card = this.players[index].cards.pop();
          console.log(card);
          console.log("Card above");
          if (card) {
            console.log("folding card");
            this.selectedRoom.folded_deck.push(card);
            cardFolded = true;
          }
        }
        if (cardFolded) {
          this.updateTable();
          this.players[index].folded = true;
          this.updatePlayer(this.players[index]);
        }
      }
    },
    retrieveCardsFromPlayers() {
      for (let c = 0; c < this.playerList.length; c++) {
        this.removePlayerCards(c);
      }
    },
    clearFoldedDeck() {
      this.selectedRoom.folded_deck = [];
    },
    clearCommunityCard() {
      this.selectedRoom.community_cards = [];
    },
    clearDeck() {
      this.selectedRoom.deck = [];
    },
    dealCards() {
      console.log(this.smallBlindNumber);
      console.log(this.playerList.length);
      console.log("Dealing Cards");
      let lengthNumber = this.playerList.length + this.smallBlindNumber;
      for (let c = this.smallBlindNumber; c < lengthNumber; c++) {
        if (c >= this.playerList.length) {
          // players = 5
          // c = 4
          // 0 hasnt been done as small blind = 3
          //
          let playerIndex = Math.abs(this.playerList.length - c);
          setTimeout(
            () => {
              console.log(this.players[playerIndex]);

              if (typeof this.players[playerIndex].cards == "string") {
                this.players[playerIndex].cards = JSON.parse(
                  this.players[playerIndex].cards
                );
                if (typeof this.players[playerIndex].cards == "string") {
                  this.players[playerIndex].cards = [];
                }
              }
              console.log(this.players[playerIndex]);
              if (
                this.players[playerIndex].folded == false &&
                this.players[playerIndex].chips > 0
              ) {
                let cardToGive = this.selectedRoom.deck.pop();
                this.players[playerIndex].cards.push(cardToGive);
                this.updateTable();
                this.updatePlayer(this.players[playerIndex]);
              }
            },
            c * 500,
            playerIndex
          );
        } else {
          let playerIndex = c;
          setTimeout(
            () => {
              console.log(this.players[playerIndex]);
              if (typeof this.players[playerIndex].cards == "string") {
                this.players[playerIndex].cards = JSON.parse(
                  this.players[playerIndex].cards
                );
                if (typeof this.players[playerIndex].cards == "string") {
                  this.players[playerIndex].cards = [];
                }
              }
              console.log(this.players[playerIndex]);
              if (
                this.players[playerIndex].folded == false &&
                this.players[playerIndex].chips > 0
              ) {
                let cardToGive = this.selectedRoom.deck.pop();
                this.players[playerIndex].cards.push(cardToGive);
                this.updateTable();
                this.updatePlayer(this.players[playerIndex]);
              }
            },
            c * 500,
            playerIndex
          );
        }
      }

      setTimeout(() => {
        if (this.firstCardDeal) {
          this.firstCardDeal = false;
          this.dealCards();
        } else {
          this.firstCardDeal = true;
        }
      }, this.playerList.length * 500);
    },
    async startGame() {
      await this.getDeck();
      await this.shuffleDeck();
      this.selectedRoom.game_started = true;
      this.updateTable();
    },
    async revealCards(amount, type) {
      this.selectedRoom.game_status = type;
      this.selectedRoom.minimum_bet = this.selectedRoom.big_blind;
      this.selectedRoom.raise_difference = this.selectedRoom.big_blind;
      this.selectedRoom.entire_pot =
        parseInt(this.selectedRoom.entire_pot) +
        parseInt(this.selectedRoom.round_pot);
      this.selectedRoom.round_pot = 0;
      let burnCard = this.selectedRoom.deck.pop();
      if (burnCard) {
        this.selectedRoom.folded_deck.push(burnCard);
      }
      this.updateTable();
      for (let c = 0; c < this.players.length; c++) {
        this.players[c].pot = 0;
        this.players[c].betting_active = false;
        this.updatePlayer(this.players[c]);
      }
      setTimeout(() => {
        for (let c = 0; c < amount; c++) {
          let card = this.selectedRoom.deck.pop();
          if (card) {
            this.selectedRoom.community_cards.push(card);
            this.updateTable();

            this.startBettingRound();
          }
        }
      }, 2000);
    },
    async endRound() {
      console.log("Ending Round");
      this.selectedRoom.game_status = "ending";
      this.updateTable();
      let hands = [];
      for (let c = 0; c < this.playerList.length; c++) {
        if (this.players[c].folded == false) {
          let playerHand = this.reviewHand(c);
          playerHand.player = this.players[c];
          console.log(playerHand);
          hands.push(playerHand);
        }
      }
      let winners = Hand.winners(hands);
      console.log("Winner Below");
      console.log(winners);
      let winningSplit = winners.length;

      for (let c = 0; c < winners.length; c++) {
        let findPlayer = this.players.findIndex(player => {
          if (player.id == winners[c].player.id) {
            return player;
          }
        });
        this.players[findPlayer].chips =
          parseInt(this.players[findPlayer].chips) +
          parseInt(this.selectedRoom.entire_pot) / winningSplit;
      }
      this.setTableEntirePot(0);
      this.updateTable();
      this.startRound();
    },
    async startRound() {
      this.selectedRoom.game_status = "preflop";
      this.roundNumber = 0;
      this.retrieveCardsFromPlayers();
      this.clearCommunityCard();
      this.clearFoldedDeck();
      this.clearDeck();
      this.getDeck();
      this.shuffleDeck();
      this.setTableRoundPot(0);
      this.setTableMinimumBet(this.selectedRoom.big_blind);
      this.setTableLatestRaise(this.selectedRoom.big_blind);
      this.setTableRaiseDifference(this.selectedRoom.big_blind);
      this.updateTable();
      let playersAllowed = this.players.filter(player => {
        if (player.chips > 0) {
          return player;
        }
      });

      if (this.dealerNumber < playersAllowed.length - 1) {
        this.dealerNumber++;
      } else {
        this.dealerNumber = 0;
      }
      let anticipatedBigBlind = this.dealerNumber + 2;

      let anticipatedSmallBlind = this.dealerNumber + 1;
      if (anticipatedSmallBlind >= playersAllowed.length) {
        anticipatedSmallBlind = 0;
        anticipatedBigBlind = 1;
      } else {
        if (anticipatedBigBlind >= playersAllowed.length) {
          anticipatedBigBlind = 0;
        }
      }
      this.bigBlindNumber = anticipatedBigBlind;
      this.smallBlindNumber = anticipatedSmallBlind;

      for (let c = 0; c < playersAllowed.length; c++) {
        playersAllowed[c].folded = false;
        playersAllowed[c].pot = 0;
        playersAllowed[c].round_pot = 0;

        if (this.dealerNumber == c) {
          playersAllowed[c].dealer = true;
        } else {
          playersAllowed[c].dealer = false;
        }

        if (anticipatedSmallBlind == c) {
          playersAllowed[c].small_blind = true;
          this.playerBet(this.selectedRoom.small_blind, c, true);
        } else {
          playersAllowed[c].small_blind = false;
        }

        if (anticipatedBigBlind == c) {
          playersAllowed[c].big_blind = true;
          this.playerBet(this.selectedRoom.big_blind, c, true);
        } else {
          playersAllowed[c].big_blind = false;
        }
        this.updatePlayer(playersAllowed[c]);
      }

      await this.dealCards();
      this.startBettingRound();
    },
    playerBet(value, playerIndex, automatic = false) {
      this.players[playerIndex].chips = this.players[playerIndex].chips - value;

      this.players[playerIndex].pot =
        parseInt(this.players[playerIndex].pot) + parseInt(value);
      if (automatic == false) {
        this.players[playerIndex].last_bet = value;
      }
      this.players[playerIndex].round_pot =
        parseInt(this.players[playerIndex].round_pot) + parseInt(value);

      this.selectedRoom.round_pot =
        parseInt(this.selectedRoom.round_pot) + parseInt(value);

      this.players[playerIndex].betting_active = false;

      this.updatePlayer(this.players[playerIndex]);
      let playersAllowed = this.players.filter(player => {
        if (player.chips > 0 && player.folded == false) {
          return player;
        }
      });
      let nextPlayer = 0;

      if (playerIndex < playersAllowed.length - 1) {
        nextPlayer = playerIndex + 1;
      } else {
        nextPlayer = 0;
      }

      if (this.players[nextPlayer].pot == this.selectedRoom.minimum_bet) {
        console.log("Player has done the minimum bet");
        let communityCardsLength = this.selectedRoom.community_cards.length;
        if (communityCardsLength == 0) {
          this.revealCards(3, "flop");
        } else if (communityCardsLength == 3) {
          this.revealCards(1, "fold");
        } else if (communityCardsLength == 4) {
          this.revealCards(1, "river");
        } else {
          console.log("Game finished");
          this.endRound();
        }
      } else {
        console.log("player hasn't done minumum bet");
        this.players[nextPlayer].betting_active = true;
        this.updatePlayer(this.players[nextPlayer]);
      }
      this.updateTable();
    },
    setTableRoundPot(amount) {
      this.selectedRoom.round_pot = amount;
    },
    setTableEntirePot(amount) {
      this.selectedRoom.entire_pot = amount;
    },
    setTableMinimumBet(amount) {
      this.selectedRoom.minimum_bet = amount;
    },
    setTableLatestRaise(amount) {
      this.selectedRoom.latest_raise = amount;
    },
    setTableRaiseDifference(amount) {
      this.selectedRoom.raise_difference = amount;
    },
    async startBettingRound() {
      let personToStartBetting = null;
      let playersAllowed = this.players.filter(player => {
        if (player.chips > 0 && player.folded == false) {
          return player;
        }
      });
      let playerToStart = 0;
      if (this.selectedRoom.game_status == "preflop") {
        console.log("Pre flop");
        // Start on after big blind
        let findPlayer = playersAllowed.findIndex(player => {
          if (player.big_blind == true) {
            return player;
          }
        });
        console.log(findPlayer);
        if (findPlayer < playersAllowed.length - 1) {
          playerToStart = findPlayer + 1;
        } else {
          playerToStart = 0;
        }
      } else {
        console.log("After pre flop");
        // Start on small Blind
        let findPlayer = playersAllowed.findIndex(player => {
          if (player.small_blind == true) {
            return player;
          }
        });
        console.log(findPlayer);
        playerToStart = findPlayer;
      }

      let oldPlayerActive = this.players.findIndex(player => {
        if (player.betting_active == true) {
          return player;
        }
      });
      let playerToUpdateIndex = this.players.findIndex(player => {
        if (player.id == playersAllowed[playerToStart].id) {
          return player;
        }
      });

      if (oldPlayerActive >= 0 && playerToUpdateIndex !== oldPlayerActive) {
        console.log("old player active lets remove him");
        console.log(this.players[oldPlayerActive]);
        this.players[oldPlayerActive].betting_active = false;
        this.updatePlayer(this.players[oldPlayerActive]);
      }

      console.log(this.players[playerToUpdateIndex]);
      this.players[playerToUpdateIndex].betting_active = true;
      console.log("player to start: " + playerToStart);
      console.log(
        "player to start player to update idnex: " + playerToUpdateIndex
      );
      console.log(playersAllowed[playerToStart]);
      console.log(this.players[playerToUpdateIndex]);
      this.updatePlayer(this.players[playerToUpdateIndex]);
    },
    getDeck() {
      var deck = new Array();

      for (var i = 0; i < this.suits.length; i++) {
        for (var x = 0; x < this.values.length; x++) {
          var card = { value: this.values[x], suit: this.suits[i] };
          deck.push(card);
        }
      }
      this.selectedRoom.deck = deck;
    },
    shuffleDeck(updateTable = false) {
      for (var i = 0; i < 1000; i++) {
        var location1 = Math.floor(
          Math.random() * this.selectedRoom.deck.length
        );
        var location2 = Math.floor(
          Math.random() * this.selectedRoom.deck.length
        );
        var tmp = this.selectedRoom.deck[location1];

        this.selectedRoom.deck[location1] = this.selectedRoom.deck[location2];
        this.selectedRoom.deck[location2] = tmp;
      }
      if (updateTable) {
        this.updateTable();
      }
    }
  },
  mounted() {}
};
</script>
<style>
.card {
  background: #fff;
  max-width: 160px;
  max-height: 222px;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 25px;
  height: 100vh;
  width: 100vw;
  box-shadow: 0px 0px 4px rgba(0, 0, 0, 0.5);
  border-radius: 7px;
}

.top__left {
  top: 0;
  left: 0;
}

.icon svg {
  max-width: 35px;
}
.icon {
  display: flex;
  align-items: center;
  justify-content: center;
  box-sizing: border-box;
  padding: 5px;
}

.bottom__right {
  bottom: 0;
  right: 0;
  transform: rotate(180deg);
}

.icon svg {
  width: 100%;
  height: 100%;
}

.diagram {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-wrap: wrap;
  width: 100%;
  height: 100%;
}

.cards .card {
  margin: 15px;
}

.bottom__right .icon,
.top__left .icon {
  max-width: 20px;
}

h3.value {
  margin: 0 0 5px;
}

.bottom__right,
.top__left {
  padding: 15px;
  display: flex;
  flex-direction: column;
  align-items: center;
  position: absolute;
  justify-content: center;
}
.spades,
.clubs,
.spades svg,
.clubs svg {
  color: #000000;
  fill: #000000;
}

.hearts,
.diamonds,
.hearts svg,
.diamonds svg {
  color: red;
  fill: red;
}

.value--2 .diagram,
.value--3 .diagram {
  flex-direction: column;
  justify-content: space-between;
}

.value--2 .diagram .icon:nth-child(2),
.value--3 .diagram .icon:nth-child(3),
.value--4 .diagram .icon:nth-child(n + 3),
.value--5 .diagram .icon:nth-child(n + 3),
.value--7 .diagram .icon:nth-child(n + 6),
.value--8 .diagram .icon:nth-child(n + 6),
.value--10 .diagram .icon:nth-child(3),
.value--10 .diagram .icon:nth-child(4),
.value--10 .diagram .icon:nth-child(6),
.value--10 .diagram .icon:nth-child(9),
.value--10 .diagram .icon:nth-child(10),
.value--6 .diagram .icon:nth-child(3),
.value--6 .diagram .icon:nth-child(6),
.value--9 .diagram .icon:nth-child(3),
.value--9 .diagram .icon:nth-child(4),
.value--9 .diagram .icon:nth-child(8),
.value--9 .diagram .icon:nth-child(9) {
  transform: rotate(180deg);
}

.value--4 .diagram .icon {
  width: 50%;
}

.value--5 .diagram .icon:nth-child(3) {
  width: 100%;
}

.value--7 .diagram .icon {
  width: 50%;
}

.value--7 .diagram .icon:nth-child(3) {
  width: 100%;
  margin: -25px 0;
}

.value--10 .diagram,
.value--6 .diagram,
.value--10 .diagram,
.value--9 .diagram {
  flex-direction: column;
}

.value--10 .diagram .icon:nth-child(5),
.value--10 .diagram .icon:nth-child(6) {
  height: 50%;
  margin: 0 -25px;
}

.value--6 .diagram .icon {
  height: 33%;
}

.value--9 .diagram .icon:nth-child(5) {
  height: 100%;
  margin: 0 -25px;
}

.value--8 .diagram .icon:nth-child(3),
.value--8 .diagram .icon:nth-child(6) {
  width: 100%;
  margin: -25px 0;
}

.hearts .diagram,
.diamonds .diagram {
  border-color: red;
}

.deck .card {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}

.deck .card:last-of-type {
  position: relative;
}

.deck {
  display: flex;
  position: relative;
}

.deck .card:not(:last-child) {
  box-shadow: none;
}

.front,
.back {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  backface-visibility: hidden;
  padding: 25px;
  box-sizing: border-box;
  -webkit-backface-visibility: hidden;
}
.back .pattern {
  background-image: url("~/assets/patterns/checkerboard-cross.png");
}

.card {
  transform-style: preserve-3d;
  transition: transform 1s;
}

.card .back {
  transform: rotateY(180deg);
  background: #fff;
}

.card.flipped {
  transform: rotateY(180deg);
}

.cards {
}
.pattern {
  width: 100%;
  height: 100%;
  background-size: 150px;
}

.card .back {
  border-radius: 7px;
  padding: 15px;
}

.pattern {
}
.duo {
  display: flex;
}
.deck {
  display: table;
  border: 2px solid #000000;
  border-radius: 7px;
  border-bottom-width: 20px;
  border-bottom-left-radius: 15px;
  border-bottom-right-radius: 15px;
}
.diagram img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  object-position: top;
}

.diagram {
  border: 1px solid #000;
  box-sizing: border-box;
}

.bottom__right,
.top__left {
  background: #fff;
  padding: 5px;
  margin: 11px;
}
.players {
  display: flex;
}
.players .player {
  margin-right: 15px;
  max-width: 25vw;
}
.player.currently--active {
  border-top: 2px solid red;
}

.player.device--player {
  border-bottom: 2px solid green;
}
</style>
