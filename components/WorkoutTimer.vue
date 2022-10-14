<template lang="pug">
.d-flex.flex-row.justify-content-center
  .p-2
    table
      tr
        td Rounds
        td
          input.form-control-sm(type="text" min=0 max=1000 v-model="rounds" :disabled='active').text-end
      tr
        td Exercises per Round
        td
          input.form-control-sm(type="text" min=0 max=1000 v-model="movements" :disabled='active').text-end
      tr
        td Work Time (sec.)
        td
          input.form-control-sm(type="text" min=0 max=1000 v-model="work" :disabled='active').text-end
      tr
        td Rest Time (sec.)
        td
          input.form-control-sm(type="text" min=0 max=1000 v-model="rest" :disabled='active').text-end
      tr
        td Lead-in time (sec.)
        td
          input.form-control-sm(type="text" min=0 max=1000 v-model="leadIn" :disabled='active').text-end
      tr
        td Completed rounds
        td.text-end {{ completedRounds }}
      tr
        td Completed exercises this round
        td.text-end {{ completedMovements }}
      tr
        td Estimated time
        td.text-end {{ estimate | toHMS }}
      tr
        td Elapsed time
        td.text-end {{ elapsedTime | toHMS }}
      tr
        td Time remaining
        td.text-end {{ (estimate - elapsedTime) | toHMS }}
      tr
        td Current countdown
        td.text-end {{ timerCountdown | toHMS }}
    .d-flex.flex-row.justify-content-between
      button.btn.btn-success(@click='start' :disabled='active') Start
      button.btn.btn-danger(@click='stop' :disabled='!active') Stop
    .d-flex.flex-row.justify-content-center
      .fs-1 {{ message }}
    audio(src='beep-01a.wav' preload='auto')#beep-01a
    audio(src='beep-02.wav' preload='auto')#beep-02
    audio(src='giddy-up.wav' preload='auto')#giddy-up
    audio(src='oh-yeah.mp3' preload='auto')#oh-yeah

    .d-flex.flex-row.justify-content-center
      svg(:height='radius * 2' :width='radius * 2')
        circle(
          class="gauge_base"
          :cx='radius'
          :cy='radius'
          fill="transparent"
          :r='radius - (strokeWidth / 2)'
          stroke="rgba(1, 1, 1, .20)"
          :stroke-width='strokeWidth'
          )
        circle(
          class="gauge_base"
          :cx='radius'
          :cy='radius'
          fill="transparent"
          :r='radius - (strokeWidth / 2)'
          stroke="rgba(.1, .1, .1, .50)"
          :transform='`rotate(-90, ${radius}, ${radius})`'
          :stroke-dasharray="dashArray"
          :stroke-width='strokeWidth'
          )
        text(x="50%" y="53%"
          font-family='Consolas'
          dominant-baseline="middle" text-anchor="middle" font-size='7rem') {{ timerCountdown }}
</template>

<script lang="ts">
import Vue from 'vue';
import NoSleep from 'nosleep.js';
import 'bootstrap/dist/css/bootstrap.css';

const noSleep = new NoSleep();

export default Vue.extend({
  name: 'WorkoutTimer',
  filters: {
    toHMS(tsec: number) {
      const hours = Math.trunc(tsec / 3600);
      const minutes = Math.trunc((tsec - hours * 3600) / 60);
      const seconds = tsec - (hours * 3600 + minutes * 60);
      return (
        hours.toString().padStart(2, '0') +
        ':' +
        minutes.toString().padStart(2, '0') +
        ':' +
        seconds.toString().padStart(2, '0')
      );
    },
  },
  data() {
    return {
      rounds: '1',
      movements: '1',
      work: '20',
      rest: '10',
      leadIn: '10',

      radius: 100,
      strokeWidth: 20,
      dashArray: '',

      timerCountdown: 0,
      elapsedTime: 0,
      estimate: 0,
      completedRounds: 0,
      completedMovements: 0,
      active: false,
      timerActive: undefined as NodeJS.Timer | undefined,
      timerElapsed: undefined as NodeJS.Timer | undefined,
      message: 'Press Start',
      dial: undefined,
    };
  },
  watch: {
    rounds() {
      this.computeEstimate();
    },
    movements() {
      this.computeEstimate();
    },
    work() {
      this.computeEstimate();
    },
    rest() {
      this.computeEstimate();
    },
  },
  mounted() {
    this.computeEstimate();
  },
  methods: {
    setGauge(at: number, total: number) {
      const percent = at / total;
      const innerRadius = this.radius - this.strokeWidth / 2;
      const circumference = innerRadius * 2 * Math.PI;
      const arc = circumference * percent;
      const dashArray = `${arc} ${circumference}`;
      this.dashArray = dashArray;
    },
    computeEstimate() {
      this.estimate =
        parseInt(this.rounds) *
        parseInt(this.movements) *
        (parseInt(this.work) + parseInt(this.rest));
    },
    main_beep() {
      const x = document.getElementById('beep-01a') as HTMLAudioElement;
      if (x) {
        x.play();
      }
    },
    heads_up_beep() {
      const x = document.getElementById('beep-02') as HTMLAudioElement;
      if (x) {
        x.play();
      }
    },
    start_workout() {
      const x = document.getElementById('giddy-up') as HTMLAudioElement;
      if (x) {
        x.play();
      }
    },
    end_workout() {
      const x = document.getElementById('oh-yeah') as HTMLAudioElement;
      if (x) {
        x.play();
      }
    },
    startActiveTimer(t: number): Promise<void> {
      if (t <= 0) {
        return Promise.resolve();
      }
      this.timerCountdown = t;
      this.setGauge(0, t);
      return new Promise<void>((resolve) => {
        this.timerActive = setInterval(() => {
          this.timerCountdown -= 1;
          this.setGauge(t - this.timerCountdown, t);
          if (this.timerCountdown === 0) {
            clearInterval(this.timerActive);
            this.timerActive = undefined;
            return resolve();
          }
          // Gotta be a sneaky way to to do this, man!
          if (
            this.timerCountdown === 3 ||
            this.timerCountdown === 2 ||
            this.timerCountdown === 1
          ) {
            this.heads_up_beep();
          }
        }, 1000);
      });
    },
    async start() {
      if (this.active === true) {
        return;
      }
      document.addEventListener(
        'click',
        function enableNoSleep() {
          document.removeEventListener('click', enableNoSleep, false);
          noSleep.enable();
        },
        false
      );
      this.main_beep();
      this.message = 'Get Ready';
      this.completedRounds = 0;
      this.completedMovements = 0;
      this.elapsedTime = 0.0;
      this.active = true;
      document.body.style.backgroundColor = 'orange';
      await this.startActiveTimer(parseInt(this.leadIn));
      this.timerElapsed = setInterval(() => {
        this.elapsedTime += 1;
      }, 1000);
      this.start_workout();
      for (let r = 0; r < parseInt(this.rounds); ++r) {
        for (let m = 0; m < parseInt(this.movements); ++m) {
          if (r !== 0 || m !== 0) {
            this.main_beep();
          }
          document.body.style.backgroundColor = 'lightgreen';
          this.message = `Workout #${m + 1}`;
          await this.startActiveTimer(parseInt(this.work));
          this.main_beep();
          document.body.style.backgroundColor = 'orange';
          this.message = 'Rest';
          await this.startActiveTimer(parseInt(this.rest));
          ++this.completedMovements;
        }
        ++this.completedRounds;
      }
      this.completedMovements = 0;
      clearInterval(this.timerElapsed);
      this.active = false;
      document.body.style.backgroundColor = '#eee';
      this.end_workout();
      this.message = 'Done';
      noSleep.disable();
    },
    stop() {
      if (this.active === false) {
        return;
      }
      noSleep.disable();
      this.setGauge(0, 100);
      this.message = 'Press Start';
      this.main_beep();
      clearInterval(this.timerActive);
      clearInterval(this.timerElapsed);
      this.timerCountdown = 0;
      this.active = false;
      document.body.style.backgroundColor = '#eee';
    },
  },
});
</script>

<style>
body {
  background-color: #eee;
}
</style>
