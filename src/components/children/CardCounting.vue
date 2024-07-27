<template>
  <div class="card-counting">
    <p>Cards left in deck: {{ cardsLeft }}</p>
    <p>Running Count: {{ runningCount }}</p>
    <p>True Count: {{ trueCount }}</p>
    <p>Play Style: {{ aggressionMeter }}</p>
    <div class="aggression-meter">
      <div class="meter">
        <div
          class="meter-bar"
          :class="aggressionMeterClass"
          :style="{ width: `${Math.abs(trueCount) * 20}%` }"
        ></div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps({
  cardsLeft: {
    type: Number,
    required: true,
  },
  runningCount: {
    type: Number,
    required: true,
  },
})

const trueCount = computed(() => {
  const decksRemaining = Math.ceil(props.cardsLeft / 52)
  return decksRemaining
    ? Math.round((props.runningCount / decksRemaining) * 100) / 100
    : 0
})

const aggressionMeter = computed(() => {
  if (trueCount.value > 0) return 'Conservative'
  if (trueCount.value < 0) return 'Aggressive'
  return 'Neutral'
})

const aggressionMeterClass = computed(() => {
  if (trueCount.value > 0) return 'conservative'
  if (trueCount.value < 0) return 'aggressive'
  return 'neutral'
})
</script>

<style scoped>
.card-counting {
  text-align: start;
}

.meter {
  width: 100%;
  height: 20px;
  background-color: #e0e0e0;
  border-radius: 10px;
  overflow: hidden;
}

.meter-bar {
  height: 100%;
  transition: width 0.5s ease-in-out;
}

.meter-bar.aggressive {
  background-color: #e74c3c;
}

.meter-bar.neutral {
  background-color: #f1c40f;
}

.meter-bar.conservative {
  background-color: #2ecc71;
}
</style>
