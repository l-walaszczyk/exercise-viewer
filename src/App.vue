<template>
  <header>
    <h1>Exercise Viewer</h1>
  </header>

  <main>
    <div class="workout-selection">
      <label for="workout-select">Workout:</label>
      <select id="workout-select" v-model="currentWorkout">
        <option v-for="workout in workouts" :key="workout.name" :value="workout">
          {{ workout.name }}
        </option>
      </select>
    </div>

    <div class="exercise-selection">
      <label for="exercise-select">Exercise:</label>
      <select id="exercise-select" v-model="currentExercise">
        <option v-for="exercise in currentWorkout.exercises" :key="exercise" :value="exercise">
          {{ formatExerciseName(exercise) }}
        </option>
      </select>

      <div class="exercise-navigation">
        <button @click="navigateToPrevExercise" :disabled="isFirstExercise" class="nav-button">
          ← Previous
        </button>
        <button @click="navigateToNextExercise" :disabled="isLastExercise" class="nav-button">
          Next →
        </button>
      </div>
    </div>

    <div class="exercise-details" v-if="currentExerciseData.exerciseName">
      <h2>{{ formatExerciseName(currentExerciseData.exerciseName) }}</h2>

      <div class="video-container" v-if="currentVideo">
        <div class="video-wrapper">
          <video controls autoplay :src="currentVideo.video" class="main-video"></video>
        </div>
      </div>

      <div class="thumbnails-container">
        <div v-for="(video, index) in currentExerciseVideos" :key="index" class="thumbnail"
          :class="{ active: currentVideoIndex === index }" @click="selectVideo(index)">
          <img :src="video.thumbnail" :alt="`Video ${index + 1}`" />
        </div>
      </div>
    </div>
  </main>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const workouts = ref([
  {
    name: 'Safe & Satisfying Day A',
    exercises: [
      'LUNGE/ALTERNATING_DUMBBELL_LUNGE',
      'LUNGE/DUMBBELL_REVERSE_LUNGE',
      'HIP_RAISE/BARBELL_HIP_THRUST_WITH_BENCH',
      'BENCH_PRESS/DUMBBELL_BENCH_PRESS',
      'SHOULDER_PRESS/DUMBBELL_SHOULDER_PRESS',
      'PULL_UP/LAT_PULLDOWN',
      'ROW/ONE_ARM_BENT_OVER_ROW',
      'ROW/FACE_PULL',
      'TRICEPS_EXTENSION/TRICEPS_PRESSDOWN',
      'CURL/STANDING_DUMBBELL_BICEPS_CURL',
      'PLANK/PLANK',
    ],
  },
  {
    name: 'Full Body (Focus: Squat, Push, Vertical Pull)',
    exercises: [
      'MOVE/ARM_CIRCLES_LOW_FORWARD_WHEELCHAIR',
      'WARM_UP/SHOULDER_ROLLS',
      'WARM_UP/HIP_CIRCLES',
      'SQUAT/SQUAT',
      'SQUAT/GOBLET_SQUAT',
      'BENCH_PRESS/DUMBBELL_BENCH_PRESS',
      'PULL_UP/LAT_PULLDOWN',
      'CALF_RAISE/STANDING_BARBELL_CALF_RAISE ',
      'POSE/CAT_COW',
      'POSE/CHILDS',
      'WARM_UP/STANDING_HAMSTRING_STRETCH',
      'WARM_UP/TRICEPS_STRETCH',
      'WARM_UP/BICEPS_STRETCH',
    ],
  },
  {
    name: 'Full Body (Focus: Hinge, Push, Horizontal Pull)',
    exercises: [
      'MOVE/ARM_CIRCLES_LOW_FORWARD_WHEELCHAIR',
      'WARM_UP/SHOULDER_ROLLS',
      'WARM_UP/HIP_CIRCLES',
      'LEG_CURL/GOOD_MORNING',
      'SHOULDER_PRESS/DUMBBELL_SHOULDER_PRESS',
      'ROW/CABLE_ROW_STANDING',
      'LUNGE/ALTERNATING_DUMBBELL_LUNGE',
      'POSE/CAT_COW',
      'POSE/ONE_LEGGED_PIGEON',
      'POSE/CHILDS',
      'MOVE/BABY_COBRA',
      'POSE/SUPINE_SPINAL_TWIST',
    ],
  },
])

const currentWorkout = ref(workouts.value[0])
const currentExercise = ref(currentWorkout.value.exercises[0])
const currentVideoIndex = ref(0)
const exerciseCache = ref({}) // Cache object to store fetched exercise data

const currentExerciseData = ref({
  bodyParts: '',
  description: '',
  difficulty: '',
  equipment: '',
  exerciseCategory: '',
  exerciseName: '',
  exerciseSteps: [],
  focuses: '',
  locale: 'pl-PL',
  videos: [],
  exerciseSourceId: '',
  heroImage: '',
  primaryMuscles: [],
  secondaryMuscles: [],
  tips: [],
})

const currentExerciseVideos = computed(() => {
  return currentExerciseData.value.videos.map((video) => ({
    thumbnail: `https://connectvideo.garmin.com/${video.thumbnail}`,
    video: `https://connectvideo.garmin.com/${video.video}`,
  }))
})

const currentVideo = computed(() => {
  if (currentExerciseVideos.value.length === 0) return null
  return currentExerciseVideos.value[currentVideoIndex.value]
})

// Compute if current exercise is the first one
const isFirstExercise = computed(() => {
  return currentWorkout.value.exercises.indexOf(currentExercise.value) === 0
})

// Compute if current exercise is the last one
const isLastExercise = computed(() => {
  const exercises = currentWorkout.value.exercises
  return exercises.indexOf(currentExercise.value) === exercises.length - 1
})

const fetchExerciseData = async (exerciseName) => {
  // Check if we already have this exercise data in cache
  if (exerciseCache.value[exerciseName]) {
    currentExerciseData.value = exerciseCache.value[exerciseName]
    currentVideoIndex.value = 0
    return
  }

  try {
    const response = await fetch(
      `https://api.allorigins.win/get?url=${encodeURIComponent(`https://connect.garmin.com/web-data/exercises/pl-PL/${exerciseName}.json`)}`,
    )
    if (!response.ok) {
      throw new Error('Network response was not ok')
    }
    const res = await response.json()
    const data = JSON.parse(res.contents)

    // Store in cache
    exerciseCache.value[exerciseName] = data

    // Update current data
    currentExerciseData.value = data

    // Reset video index to 0 when loading a new exercise
    currentVideoIndex.value = 0
  } catch (error) {
    console.error('Error fetching exercise data:', error)
  }
}

const formatExerciseName = (exerciseName) => {
  if (!exerciseName) return ''
  // Get the last part of the exercise name (after the last /)
  const name = exerciseName.split('/').pop()
  // Replace underscores with spaces and title case
  return name
    .replace(/_/g, ' ')
    .toLowerCase()
    .replace(/\b\w/g, (c) => c.toUpperCase())
}

const selectVideo = (index) => {
  currentVideoIndex.value = index
}

// Navigation functions for exercises
const navigateToPrevExercise = () => {
  if (isFirstExercise.value) return

  const exercises = currentWorkout.value.exercises
  const currentIndex = exercises.indexOf(currentExercise.value)
  currentExercise.value = exercises[currentIndex - 1]
}

const navigateToNextExercise = () => {
  if (isLastExercise.value) return

  const exercises = currentWorkout.value.exercises
  const currentIndex = exercises.indexOf(currentExercise.value)
  currentExercise.value = exercises[currentIndex + 1]
}

// Watch for workout changes and update the current exercise
watch(
  () => currentWorkout.value,
  (newWorkout) => {
    // Reset to first exercise when workout changes
    currentExercise.value = newWorkout.exercises[0]
  },
)

// Watch for exercise changes and fetch data
watch(
  () => currentExercise.value,
  (newExercise) => {
    fetchExerciseData(newExercise)
  },
  { immediate: true }, // Fetch data immediately on component mount
)
</script>

<style scoped>
header {
  padding: 1rem;
  background-color: #f5f5f5;
  text-align: center;
}

main {
  max-width: 900px;
  margin: 0 auto;
  padding: 1rem;
}

.workout-selection,
.exercise-selection {
  margin-bottom: 1rem;
}

select {
  width: 100%;
  padding: 0.5rem;
  margin-top: 0.25rem;
  border-radius: 4px;
  border: 1px solid #ccc;
}

.exercise-navigation {
  display: flex;
  justify-content: space-between;
  margin-top: 0.5rem;
}

.nav-button {
  cursor: pointer;
  transition: all 0.2s;
}

.nav-button:disabled {
  cursor: not-allowed;
}

.exercise-details {
  margin-top: 2rem;
}

.video-container {
  margin-bottom: 1rem;
}

.main-video {
  width: 100%;
  max-height: 500px;
  aspect-ratio: 16 / 9;
  background-color: #000;
}

.thumbnails-container {
  display: flex;
  gap: 0.5rem;
  overflow-x: auto;
  padding-bottom: 0.5rem;
  margin-bottom: 1rem;
}

.thumbnail {
  flex: 0 0 auto;
  cursor: pointer;
  border: 2px solid transparent;
  transition: all 0.2s;
}

.thumbnail.active {
  border-color: #4285f4;
  opacity: 0.7;
}

.thumbnail img {
  width: 100px;
  aspect-ratio: 16 / 9;
  object-fit: cover;
}
</style>
