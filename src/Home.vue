<template>
  <div id="poll">
    <span class="question">{{ question.text }}</span>
    <hr/>
    <div v-for="(option, index) in question.options">
      <div v-if="show_results" class="result" style="width: 25%; text-align: right; float: left;">
        <div :class="'progress-bar' + (winner(index)?' winner':'')" :style="'float: right; height: 100%; width: ' + responsePercentages()[index] * 100 + '%; background-color: ' + markers[index].color + ';'">
        <template v-if="responsePercentages()[index] > 0">{{ Math.round(responsePercentages()[index] * 100, 2) }}%</template>
        </div>
      </div>
      <div style="width: auto;" class="option">
        <span class="button-marker" :style="'background-color: ' + markers[index].color + ';'">{{ markers[index].button }}</span>
        <span class="answer-text">{{ question.options[index] }}</span>
      </div>
      <br/>
    </div>
    <hr/>
    <div id="poll-options">
      <span class="player-count">{{ Object.keys(players).length }} Players</span> | <span class="response-count">{{ Object.keys(responses).length }} Responses</span><br/>
      <label>Show Results<input type="checkbox" v-model="show_results"/></label><br/>
      <label>Anonymous Vote<input type="checkbox" v-on:change="anonymousChanged()" v-model="anonymous_vote"/></label><br/>
      <label>Use percent of total players instead of votes<input type="checkbox" v-model="total_percent"/></label><br/>
      <button v-on:click="poll_open = !poll_open">{{ poll_open?"Close Poll":"Open Poll" }}</button>
      <button v-on:click="resetResponses()">Clear Responses</button>
      <button v-on:click="question = {text:'',options:['']}">Reset Questions</button>
      <br/>
      Quick Presets:
      <button v-on:click="question.options=['True','False']">True/False</button>
      <button v-on:click="question.options=['Yes','No']">Yes/No</button>
      <button v-on:click="question.options=['Agree','Disagree']">Agree/Disagree</button>
      <hr/>
    </div>
    <div v-if="!poll_open" id="poll-edit">
      <textarea v-model="question.text" placeholder="Question Text"></textarea><br/>
      <template v-for="(option, index) in question.options">
      <button :disabled="question.options.length < 2" v-on:click="question.options.splice(index, 1)">-</button>
      <input type="text" placeholder="Response" v-model="question.options[index]"/><br/>
      </template>
      <button v-if="question.options.length < 8" v-on:click="question.options.push('')">+</button>
    </div>
  </div>
</template>

<script>
  export default {
    data() {
      return {
        question: {
          text: "",
          options: [""]
        },
        responses: {},
        players: {},
        show_results: true,
        markers: [
          {button: "up", color: "rgb(86, 180, 223)", color_hex: 0x56b4df}, // Sky blue
          {button: "down", color: "rgb(204, 121, 167)", color_hex: 0xcc79a7}, // Reddish purple
          {button: "left", color: "rgb(230, 159, 0)", color_hex: 0xe69f00}, // Orange
          {button: "right", color: "rgb(0, 158, 115)", color_hex: 0x009e73}, // Bluish green
          {button: "a", color: "rgb(240, 228, 66)", color_hex: 0xf0e442}, // Yellow
          {button: "b", color: "rgb(86, 180, 233)", color_hex: 0x56b4e9}, // Blue
          {button: "select", color: "rgb(213, 94, 0)", color_hex: 0xd55e00}, // Vermillion
          {button: "start", color: "#ffffff", color_hex: 0xffffff}, // white
        ],
        button_index: {
          up: 0,
          down: 1,
          left: 2,
          right: 3,
          a: 4,
          b: 5,
          select: 6,
          start: 7,
        },
        total_percent: false,
        anonymous_vote: false,
        poll_open: false,
      };
    },
    mounted() {
      var self = this;

      this.$wamp.subscribe('game.poll_panels1.player.join', function(args, kwargs, details) {
        self.playerJoin(args[0]);
      }, {}).then(function(s) {});

      this.$wamp.subscribe('game.poll_panels1.player.leave', function(args, kwargs, details) {
        self.playerLeave(args[0]);
      }, {}).then(function(s) {});

      this.$wamp.subscribe('game.request_register', function(args, kwargs, details) {
        self.register();
      }, {}).then(function(s) {});

      this.register();
    },
    methods: {
      resetResponses: function() {
        this.clearIndicators();
        this.responses = {};
      },
      clearIndicators: function() {
        for (var i in this.responses) {
          this.$wamp.publish('badge.' + i + '.lights_static', Array(4).fill(0));
          this.$wamp.publish('badge.' + i + '.text', [""]);
        }
      },
      anonymousChanged: function() {
        if (this.anonymous_vote) {
          this.clearIndicators();
        } else {
          for (var i in this.responses) {
            this.$wamp.publish('badge.' + i + '.lights_static', Array(4).fill(this.markers[this.responses[i]].color_hex), {});
          }
        }
      },
      winner: function(index) {
        var res = this.responseCounts();
        var max = 0, ind = -1;

        for (var i in res) {
          if (res[i] > max) {
            max = res[i];
            ind = i;
          }
        }

        return ind == index;
      },
      responseCounts: function() {
        var res = Array(this.question.options.length).fill(0);

        for (var player in this.responses) {
          res[this.responses[player]] += 1;
        }

        return res;
      },
      responsePercentages: function() {
        var res = Array(this.question.options.length).fill(0);
        var sum = 0;

        for (var player in this.responses) {
          res[this.responses[player]] += 1;
          sum += 1;
        }

        if (this.total_percent) {
          sum = Object.keys(this.players).length;
        }

        return res.map(function(v){return sum==0?0:v/sum;});
      },
      register: function() {
        var self = this;
        this.$wamp.call('game.register', ['poll_panels1'], {'sequence': 'udud'}).then(
          function(res) {
            console.log("Initializing " + res.kwargs.players.length + " existing players");
            for (var i in res.kwargs.players) {
              self.playerJoin(res.kwargs.players[i]);
            }
          }
        );
      },
      playerJoin: function(player) {
        this.players[player] = {selection: -1, subscriptions: []};

        var sub = 'badge.' + player + '.button.press';
        var self = this;

        this.$wamp.subscribe(sub, function(args, kwargs, details) {
          self.onButton(kwargs.badge_id, args[0]);
        }, {}).then(function(s) {
          players[player].subscriptions.push(s.id);
        });
      },
      playerLeave: function(player) {
        if (this.players[player]) {
          var subs = self.players[player].subscriptions;
          for (var i in subs) {
            this.$wamp.unsubscribe(subs[i]).then(function(s) {
            });
          }
        }
      },
      onButton: function(player, b) {
        if (!this.poll_open) return;

        var i = this.button_index[b];
        if (i < this.question.options.length) {
          this.responses[player] = i;
          this.$forceUpdate();
          if (!this.anonymous_vote) {
            this.$wamp.publish('badge.' + player + '.lights_static', Array(4).fill(this.markers[i].color_hex), {});
            this.$wamp.publish('badge.' + player + '.text', [this.question.options[i]], {});
          }
        }
      }
    }
  }
</script>
