<template>
  <div>

    <!-- Sequencer -->
    <div id="digit0" class="seq_div">
    </div>
    <div id="digit1" class="seq_div">
    </div>
    <div id="column" class="seq_div">
    </div>
    <div id="digit2" class="seq_div">
    </div>
    <div id="digit3" class="seq_div">
    </div>


  </div>
  
  <button v-on:click="test">test</button>

  
</template>

<script>
const Nexus = require("nexusui")
import * as Tone from 'tone'

export default ({
  name: "Sequencer",

  data() {
    return {
      block_size: 50,

      digit0: NaN,
      digit1: NaN,
      digit2: NaN,
      digit3: NaN,
      column: NaN,

      time0: NaN,
      time1: NaN,
      time2: NaN,
      time3: NaN,

      second: NaN,

      mute: true,
    }
  },
  
  computed: {
    digits() {
      return [this.digit0, this.digit1, this.digit2, this.digit3]
    },

    times() {
      return [this.time0, this.time1, this.time2, this.time3]
    },
  },

  methods: {
    test() {
      Tone.start()
      this.mute = false
      console.log(123)
      // console.log(this.digit0.matrix.pattern)

    },

    // Set the sequencer digits to new_digits eg.[12:23]
    set_matrix_time(new_digits) {
      const digit_mats = {
        0: [[1,1,1],[1,0,1],[1,0,1],[1,0,1],[1,1,1]],
        1: [[0,1,0],[1,1,0],[0,1,0],[0,1,0],[1,1,1]],
        2: [[1,1,1],[0,0,1],[1,1,1],[1,0,0],[1,1,1]],
        3: [[1,1,1],[0,0,1],[1,1,1],[0,0,1],[1,1,1]],
        4: [[1,0,1],[1,0,1],[1,1,1],[0,0,1],[0,0,1]],
        5: [[1,1,1],[1,0,0],[1,1,1],[0,0,1],[1,1,1]],
        6: [[1,1,1],[1,0,0],[1,1,1],[1,0,1],[1,1,1]],
        7: [[1,1,1],[0,0,1],[0,1,0],[1,0,0],[1,0,0]],
        8: [[1,1,1],[1,0,1],[1,1,1],[1,0,1],[1,1,1]],
        9: [[1,1,1],[1,0,1],[1,1,1],[0,0,1],[1,1,1]],
      }
      
      for (let i=0; i<this.digits.length; i++){
        this.digits[i].matrix.set.all(digit_mats[new_digits[i]])
      }
    }
  },

  mounted() {
    // Set up instruments
    var noise = new Tone.NoiseSynth().toDestination()
    var melody = new Tone.Synth().toDestination()

    // Set up the sequencer blocks
    const seq_properties = {
        'size': [3*this.block_size,5*this.block_size],
        'mode': 'toggle',
        'rows': 5,
        'columns': 3,
      }
    const col_properties = {
        'size': [1*this.block_size,5*this.block_size],
        'mode': 'toggle',
        'rows': 5,
        'columns': 1,
      }

    this.digit0 = new Nexus.Sequencer('#digit0', seq_properties)
    this.digit1 = new Nexus.Sequencer('#digit1', seq_properties)
    this.digit2 = new Nexus.Sequencer('#digit2', seq_properties)
    this.digit3 = new Nexus.Sequencer('#digit3', seq_properties)
    this.column = new Nexus.Sequencer('#column', col_properties)
    
    this.column.matrix.set.column(0,[0,1,0,1,0])

    // Change the color scheme for the column
    // this.column.colorize("accent", "#873e23")
    this.column.colorize("fill", "#ffffff")

    // Update according to current time
    const date = new Date
    const hours = date.getHours()
    const minutes = date.getMinutes()
    this.second = date.getSeconds()

    this.time1 = hours % 10
    this.time0 = (hours - this.time1) / 10
    this.time3 = minutes % 10
    this.time2 = (minutes - this.time3) / 10
    
    this.set_matrix_time(this.times)

    // Set up the interval for sequencing
    // We use only the first sequencer
    const interval = this.digit0.interval
    interval.ms(1000)
    interval.event = () => {

      // Update this.time... yes, this is ugly
      this.second += 1
      if (this.second == 60){
        this.second = 0;
        this.time3 += 1
        if (this.time3 == 10){
          this.time3 = 0
          this.time2 += 1
          if (this.time2 == 6){
            this.time2 = 0
            this.time1 += 1
            if (this.time1 == 10){
              this.time1 = 0
              this.time0 += 1
              if (this.time0 == 3){
                this.time0 = 0
              }
            }
          }
        }
        this.set_matrix_time(this.times)
      }

      // Column pixel animation
      if (this.second % 3 == 0){
        this.column.matrix.set.column(0,[0,1,0,1,0])
        if (! this.mute)
        noise.triggerAttack("+0",0.3)
        this.column.matrix.toggle.cell(0,1)
        this.column.matrix.toggle.cell(0,3)
        setTimeout(() => {
        this.column.matrix.toggle.cell(0,1)
        this.column.matrix.toggle.cell(0,3)
        },200)
      }
      else if (this.second % 3 == 1){
         this.column.matrix.set.column(0,[0,0,0,0,0])
        setTimeout(() => {
          this.column.matrix.set.column(0,[0,1,0,0,0])
        },200)
        if (! this.mute)
        noise.triggerAttack("+0",0.05)
      }
      else if (this.second % 3 == 2){
         this.column.matrix.set.column(0,[0,0,0,0,0])
        setTimeout(() => {
          this.column.matrix.set.column(0,[0,0,0,1,0])
        },200)
        if (! this.mute)
        noise.triggerAttack("+0",0.05)
      }

      // Handle melody (second)
      // Every 6 seconds is matched to the notes of the A minor pentatonic scale (plus the octave)
      // For ever 12 seconds, the first half is ascendind the the second half, descending
      const pitch_idx = this.second % 12
      let digit_idx = Math.floor(pitch_idx / 3)
      if (this.digits[digit_idx].matrix.pattern[Math.floor(this.second/12)][pitch_idx%3] == 1 && ! this.mute){
        let pitches = ['A3','C4','D4','A3','C4','D4','E4','G4','A4','E4','G4','A4']
        melody.triggerAttack(pitches[pitch_idx])
      }
      // Make the current block blink
      this.digits[digit_idx].matrix.toggle.cell(pitch_idx%3,Math.floor(this.second/12))
      setTimeout(() => {
        this.digits[digit_idx].matrix.toggle.cell(pitch_idx%3,Math.floor(this.second/12))
      },200)
    }
    interval.start()
  }

})
</script>

<style scoped>

.seq_div{
  display:inline-block;
  padding: 10px;
}

</style>
