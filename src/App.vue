<script setup lang="ts">
import {computed, onMounted, reactive, ref, watch} from 'vue'

type MODE_T = "TIMER" | "SCORE" | "MATCH";
type STATE_T = "IDLE" | "COUNTDOWN" | "RUNNING"

const stageData = {
  1: {name: "Automatic", duration: 30},
  2: {name: "Manual", duration: 100},
  3: {name: "Modification", duration: 60},
  4: {name: "Final", duration: 90},
}

interface MatchVersus {
  redTeam: {
    team1Name: string;
    team2Name: string;
  } | {};
  blueTeam: {
    team1Name: string;
    team2Name: string;
  } | {};
}

interface SCOREELEMENT {
  name: string;
  pts: number;
  count: number;
}

interface IMATCHENTRY {
  redTeam: {
    team1Name: string;
    team2Name: string;
  };
  blueTeam: {
    team1Name: string;
    team2Name: string;
  };
  matchNumber: number;
  timestamp: number;
}

interface IAPPSTATE {
  mode: MODE_T;
  stage: number;
  redTeam: {
    team1Name: string;
    team2Name: string;
    score: number;
    elements: SCOREELEMENT[];
    penalty: number;
  };
  blueTeam: {
    team1Name: string;
    team2Name: string;
    score: number;
    elements: SCOREELEMENT[];
    penalty: number;
  };
  timer: number;
  match: Array<MatchVersus>;
  history: IHISTORYENTRY[];
  matchData: IMATCHENTRY[];
}

interface IHISTORYENTRY {
  redTeam: {
    team1Name: string;
    team2Name: string;
    score: number;
    penalty: number;
  };
  blueTeam: {
    team1Name: string;
    team2Name: string;
    score: number;
    penalty: number;
  };
  timestamp: number;
}

const defaultElements = (): SCOREELEMENT[] => [
  {name: "Balls", pts: 10, count: 0},
  {name: "Block", pts: 30, count: 0},
  {name: "Cone", pts: 30, count: 0},
  {name: "Flag hanging", pts: 50, count: 0},
  {name: "Pin Lower", pts: 10, count: 0},
  {name: "Pin Upper", pts: 20, count: 0},
  {name: "Pin", pts: 30, count: 0},
  {name: "MakeX", pts: 30, count: 0},
]

const mode = ref("TIMER" as MODE_T);
const stage = ref(1 as number);
const timer = ref(0);
const state = ref("IDLE" as STATE_T)

const redTeam1Name = ref("");
const redTeam2Name = ref("");
const redElements = ref<SCOREELEMENT[]>(defaultElements());
const redPenalty = ref(0);

const blueTeam1Name = ref("");
const blueTeam2Name = ref("");
const blueElements = ref<SCOREELEMENT[]>(defaultElements());
const bluePenalty = ref(0);

const penaltyStopThreshold = 3;

const clampPenalty = (value: number) => Math.max(0, Math.min(value, penaltyStopThreshold));

const history = ref<IHISTORYENTRY[]>([]);
const showHistory = ref(false);

// Match Data page state
const matchData = ref<IMATCHENTRY[]>([]);
const matchForm = reactive({
  matchNumber: 1,
  redTeam1: "",
  redTeam2: "",
  blueTeam1: "",
  blueTeam2: "",
});
const showDeleteModal = ref(false);
const deleteIndex = ref<number | null>(null);

const penaltyPoints = 20;
const redPenaltyTotal = computed(() => clampPenalty(redPenalty.value) * penaltyPoints);
const bluePenaltyTotal = computed(() => clampPenalty(bluePenalty.value) * penaltyPoints);
const redScore = computed(() => redElements.value.reduce((sum, el) => sum + el.pts * el.count, 0) - redPenaltyTotal.value);
const blueScore = computed(() => blueElements.value.reduce((sum, el) => sum + el.pts * el.count, 0) - bluePenaltyTotal.value);

watch([redPenalty, bluePenalty], () => {
  if (redPenalty.value >= penaltyStopThreshold || bluePenalty.value >= penaltyStopThreshold) {
    clearTimer();
    state.value = "IDLE";
  }
});

const redTeamMatch = reactive<Array<Pick<MatchVersus, "redTeam"> | {}>>([]);
const blueTeamMatch = reactive<Array<Pick<MatchVersus, "blueTeam"> | {}>>([]);

// Derived: next match number suggestion
const nextMatchNumber = computed(() => {
  if (matchData.value.length === 0) return 1;
  return Math.max(...matchData.value.map(m => m.matchNumber)) + 1;
});

let countdownInterval: number;
let timerInterval: number;

function startTimer() {
  state.value = "COUNTDOWN";
  timer.value = 5;

  countdownInterval = setInterval(() => {
    if (timer.value > 0) {
      timer.value--;
    } else {
      clearInterval(countdownInterval);
      state.value = "RUNNING";
      timer.value = stageData[stage.value as keyof typeof stageData].duration;
    }
  }, 1000);


  timerInterval = setInterval(() => {
    if (state.value !== "RUNNING") return;
    if (timer.value === 0) {
      state.value = "IDLE";
      stage.value === 4 ? stage.value = 1 : stage.value += 1;
      timer.value = 0;
      clearInterval(timerInterval);
      return;
    }
    timer.value -= 1;
  }, 1000)
}

function clearTimer() {
  clearInterval(countdownInterval);
  clearInterval(timerInterval);
  state.value = "IDLE";
  timer.value = 0;
}

function nextState() {
  if (stage.value === 4) {
    stage.value = 4;
  } else {
    stage.value += 1;
  }
  clearTimer();
}

function prevState() {
  if (stage.value === 1) {
    stage.value = 1;
  } else {
    stage.value -= 1;
  }
  clearTimer();
}

function saveToLocalStorage() {
  const appState: IAPPSTATE = {
    mode: mode.value,
    stage: stage.value,
    redTeam: {
      team1Name: redTeam1Name.value,
      team2Name: redTeam2Name.value,
      score: redScore.value,
      elements: redElements.value,
      penalty: redPenalty.value,
    },
    blueTeam: {
      team1Name: blueTeam1Name.value,
      team2Name: blueTeam2Name.value,
      score: blueScore.value,
      elements: blueElements.value,
      penalty: bluePenalty.value,
    },
    timer: timer.value,
    match: [{redTeam: redTeamMatch, blueTeam: blueTeamMatch}],
    history: history.value,
    matchData: matchData.value,
  };
  localStorage.setItem("appState", JSON.stringify(appState));
}

function saveMatch() {
  const entry: IHISTORYENTRY = {
    redTeam: {
      team1Name: redTeam1Name.value,
      team2Name: redTeam2Name.value,
      score: redScore.value,
      penalty: redPenalty.value,
    },
    blueTeam: {
      team1Name: blueTeam1Name.value,
      team2Name: blueTeam2Name.value,
      score: blueScore.value,
      penalty: bluePenalty.value,
    },
    timestamp: Date.now(),
  };
  history.value.push(entry);
  saveToLocalStorage();
  resetScore();
}

function resetScore() {
  redTeam1Name.value = "";
  redTeam2Name.value = "";
  redElements.value = defaultElements();
  redPenalty.value = 0;
  blueTeam1Name.value = "";
  blueTeam2Name.value = "";
  blueElements.value = defaultElements();
  bluePenalty.value = 0;
  saveToLocalStorage();
}

// Match Data: add a new match entry
function addMatchEntry() {
  if (!matchForm.redTeam1 && !matchForm.redTeam2 && !matchForm.blueTeam1 && !matchForm.blueTeam2) return;

  const entry: IMATCHENTRY = {
    matchNumber: matchForm.matchNumber,
    redTeam: {
      team1Name: matchForm.redTeam1,
      team2Name: matchForm.redTeam2,
    },
    blueTeam: {
      team1Name: matchForm.blueTeam1,
      team2Name: matchForm.blueTeam2,
    },
    timestamp: Date.now(),
  };

  matchData.value.push(entry);
  saveToLocalStorage();

  // Reset form, auto-increment match number
  matchForm.matchNumber = nextMatchNumber.value;
  matchForm.redTeam1 = "";
  matchForm.redTeam2 = "";
  matchForm.blueTeam1 = "";
  matchForm.blueTeam2 = "";
}

// Match Data: open delete confirmation
function confirmDeleteMatch(index: number) {
  deleteIndex.value = index;
  showDeleteModal.value = true;
}

// Match Data: execute delete
function deleteMatchEntry() {
  if (deleteIndex.value !== null) {
    matchData.value.splice(deleteIndex.value, 1);
    saveToLocalStorage();
  }
  showDeleteModal.value = false;
  deleteIndex.value = null;
}

onMounted(() => {
  const storage = JSON.parse(localStorage.getItem("appState") || {} as string) as IAPPSTATE;
  if (storage) {
    mode.value = storage.mode;
    stage.value = storage.stage;
    timer.value = storage.timer;
    if (storage.redTeam) {
      redTeam1Name.value = storage.redTeam.team1Name || "";
      redTeam2Name.value = storage.redTeam.team2Name || "";
      if (storage.redTeam.elements) redElements.value = storage.redTeam.elements;
      if (storage.redTeam.penalty !== undefined) redPenalty.value = storage.redTeam.penalty;
    }
    if (storage.blueTeam) {
      blueTeam1Name.value = storage.blueTeam.team1Name || "";
      blueTeam2Name.value = storage.blueTeam.team2Name || "";
      if (storage.blueTeam.elements) blueElements.value = storage.blueTeam.elements;
      if (storage.blueTeam.penalty !== undefined) bluePenalty.value = storage.blueTeam.penalty;
    }
    if (storage.match) {
      storage.match.forEach((m: MatchVersus) => {
        if (m.redTeam) redTeamMatch.push(m.redTeam);
        if (m.blueTeam) blueTeamMatch.push(m.blueTeam);
      });
    }
    ;
    if (storage.history) history.value = storage.history;
    if (storage.matchData) {
      matchData.value = storage.matchData;
      matchForm.matchNumber = nextMatchNumber.value;
    }
  }
})
</script>

<template>
  <div class="drawer min-h-screen">
    <input id="my-drawer-2" type="checkbox" class="drawer-toggle"/>

    <!-- MAIN CONTENT -->
    <div class="drawer-content flex flex-col min-h-screen">
      <div
          class="relative min-h-screen w-full overflow-hidden bg-cover bg-center"
      >

        <!-- HEADER -->
        <div class="relative z-10 flex items-center justify-between px-6 py-4">

          <div class="flex items-center gap-4">
            <div class="flex flex-row items-center justify-center gap-4 leading-none">
              <img src="/makex.png" alt="MakeX Logo" class="h-16 w-16"/>
              <img src="/acsp.png" alt="ACSP Logo" class="h-8"/>
            </div>
          </div>

          <div class="flex items-center gap-2">
            <div class="flex-none lg:hidden">
              <label for="my-drawer-2" aria-label="open sidebar" class="btn btn-square btn-ghost">
                <svg
                    xmlns="http://www.w3.org/2000/svg"
                    fill="none"
                    viewBox="0 0 24 24"
                    class="inline-block h-6 w-6 stroke-current"
                >
                  <path
                      stroke-linecap="round"
                      stroke-linejoin="round"
                      stroke-width="2"
                      d="M4 6h16M4 12h16M4 18h16"
                  ></path>
                </svg>
              </label>
            </div>

            <!-- NAV -->
            <div class="hidden flex-none lg:block ">
              <ul class="menu menu-horizontal rounded-box bg-base-100/60 backdrop-blur px-2 gap-2">
                <li class="btn btn-primary"><a @click="mode = 'TIMER'">Timer</a></li>
                <li class="btn btn-primary"><a @click="mode = 'SCORE'">Score Sheet</a></li>
                <li class="btn btn-primary"><a @click="mode = 'MATCH'">Match Data</a></li>
                <li class="btn btn-warning"><a @click="clearTimer()">Clear Timer</a></li>
              </ul>
            </div>
          </div>
        </div>

        <!-- CONTENT -->
        <div class="relative z-10 flex flex-1 items-start justify-center py-6">

          <!-- TIMER PAGE -->
          <div class="w-full max-w-2xl" v-if="mode === 'TIMER'">
            <div class="flex flex-col items-center justify-center gap-4">
              <div class="badge badge-error badge-lg px-6 py-4 text-white font-extrabold shadow">
                Stage {{ stage }}/4
              </div>

              <h1 class="text-4xl font-black tracking-tight text-black drop-shadow-sm">
                {{ stageData[stage as keyof typeof stageData].name }} Stage
              </h1>

              <div class="flex flex-col items-center gap-1">
                <h2 v-if="state !== 'IDLE'" class="text-3xl font-extrabold tabular-nums text-black">
                  {{ new Date(timer * 1000).toISOString().slice(14, 19) }}
                </h2>
                <h3 class="text-lg font-bold text-black/80">
                  {{
                    state === "IDLE" ? "Ready for next stage?" : state === "COUNTDOWN" ? "Prepare..." : "Running..."
                  }}
                </h3>
              </div>

              <div class="flex flex-col items-center gap-3">
                <button
                    class="btn btn-error btn-lg rounded-full px-10 shadow-lg"
                    @click="startTimer"
                    :disabled="state !== 'IDLE'"
                >
                  Start Stage
                </button>

                <div class="flex flex-row gap-3 items-center justify-center">
                  <button
                      class="btn btn-outline btn-error rounded-full"
                      @click="prevState()"
                      :disabled="state === 'COUNTDOWN'"
                  >
                    Previous Stage
                  </button>
                  <button
                      class="btn btn-outline btn-success rounded-full"
                      @click="nextState()"
                      :disabled="state === 'COUNTDOWN'"
                  >
                    Next Stage
                  </button>
                </div>
              </div>
            </div>
          </div>

          <!-- SCORE PAGE -->
          <div class="w-full max-w-6xl px-4" v-else-if="mode === 'SCORE'">
            <div class="flex flex-col lg:flex-row gap-4 items-stretch">

              <!-- RED ALLIANCE -->
              <div class="flex-1 rounded-3xl bg-red-500 p-5 shadow-xl text-white">
                <h2 class="text-3xl font-black tracking-tight drop-shadow mb-4 text-center">RED ALLIANCE</h2>

                <div class="bg-white/15 rounded-2xl overflow-hidden">
                  <!-- Table Header -->
                  <div class="grid grid-cols-4 gap-2 px-4 py-2 bg-white/20">
                    <div class="text-sm font-black text-white">Element</div>
                    <div class="text-sm font-black text-white text-center">Pts</div>
                    <div class="text-sm font-black text-white text-center">Count</div>
                    <div class="text-sm font-black text-white text-center">Total</div>
                  </div>

                  <!-- Table Rows -->
                  <div
                      v-for="(el, index) in redElements"
                      :key="'red-' + index"
                      class="grid grid-cols-4 gap-2 px-4 py-2 border-t border-white/20 items-center"
                  >
                    <div class="text-sm font-bold text-white">{{ el.name }}</div>
                    <div class="text-sm font-black text-white text-center">{{ el.pts }}</div>
                    <div class="flex justify-center">
                      <input
                          type="number"
                          min="0"
                          v-model.number="el.count"
                          class="w-16 rounded-lg bg-white/20 border border-white/30 text-white text-center text-sm font-black focus:outline-none focus:ring-2 focus:ring-white/50 py-1"
                      />
                    </div>
                    <div class="text-sm font-black text-white text-center tabular-nums">{{ el.pts * el.count }}</div>
                  </div>
                  <!-- Penalty Row -->
                  <div class="grid grid-cols-4 gap-2 px-4 py-2 border-t border-white/20 items-center bg-white/10">
                    <div class="text-sm font-bold text-yellow-200">Penalty</div>
                    <div class="text-sm font-black text-yellow-200 text-center">-20</div>
                    <div class="flex justify-center">
                      <input
                          type="number"
                          min="0"
                          v-model.number="redPenalty"
                          class="w-16 rounded-lg bg-white/20 border border-white/30 text-white text-center text-sm font-black focus:outline-none focus:ring-2 focus:ring-white/50 py-1"
                      />
                    </div>
                    <div class="text-sm font-black text-yellow-200 text-center tabular-nums">{{
                        -(redPenaltyTotal)
                      }}
                    </div>
                  </div>
                </div>

                <!-- Team Names -->
                <div class="mt-4">
                  <div class="text-xs font-black text-white/90 mb-1">Team name</div>
                  <input
                      type="text"
                      v-model="redTeam1Name"
                      placeholder="Team 1"
                      class="w-full rounded-xl bg-white text-black text-sm font-bold px-3 py-2 mb-2 focus:outline-none focus:ring-2 focus:ring-white/50"
                  />
                  <input
                      type="text"
                      v-model="redTeam2Name"
                      placeholder="Team 2"
                      class="w-full rounded-xl bg-white text-black text-sm font-bold px-3 py-2 focus:outline-none focus:ring-2 focus:ring-white/50"
                  />
                </div>

                <!-- Total -->
                <div class="mt-4 rounded-2xl bg-red-600 px-5 py-3 flex items-center justify-between shadow">
                  <div class="text-lg font-black text-white">TOTAL:</div>
                  <div class="text-2xl font-black text-white tabular-nums">{{ redScore }}</div>
                </div>
              </div>

              <!-- VS BADGE -->
              <div class="flex items-center justify-center">
                <div
                    class="text-4xl font-black text-transparent bg-clip-text bg-gradient-to-b from-orange-400 to-red-600 drop-shadow-lg"
                    style="text-shadow: 0 2px 8px rgba(0,0,0,0.3);">
                  VS
                </div>
              </div>

              <!-- BLUE ALLIANCE -->
              <div class="flex-1 rounded-3xl bg-blue-500 p-5 shadow-xl text-white">
                <h2 class="text-3xl font-black tracking-tight drop-shadow mb-4 text-center">BLUE ALLIANCE</h2>

                <div class="bg-white/15 rounded-2xl overflow-hidden">
                  <!-- Table Header -->
                  <div class="grid grid-cols-4 gap-2 px-4 py-2 bg-white/20">
                    <div class="text-sm font-black text-white">Element</div>
                    <div class="text-sm font-black text-white text-center">Pts</div>
                    <div class="text-sm font-black text-white text-center">Count</div>
                    <div class="text-sm font-black text-white text-center">Total</div>
                  </div>

                  <!-- Table Rows -->
                  <div
                      v-for="(el, index) in blueElements"
                      :key="'blue-' + index"
                      class="grid grid-cols-4 gap-2 px-4 py-2 border-t border-white/20 items-center"
                  >
                    <div class="text-sm font-bold text-white">{{ el.name }}</div>
                    <div class="text-sm font-black text-white text-center">{{ el.pts }}</div>
                    <div class="flex justify-center">
                      <input
                          type="number"
                          min="0"
                          v-model.number="el.count"
                          class="w-16 rounded-lg bg-white/20 border border-white/30 text-white text-center text-sm font-black focus:outline-none focus:ring-2 focus:ring-white/50 py-1"
                      />
                    </div>
                    <div class="text-sm font-black text-white text-center tabular-nums">{{ el.pts * el.count }}</div>
                  </div>
                  <!-- Penalty Row -->
                  <div class="grid grid-cols-4 gap-2 px-4 py-2 border-t border-white/20 items-center bg-white/10">
                    <div class="text-sm font-bold text-yellow-200">Penalty</div>
                    <div class="text-sm font-black text-yellow-200 text-center">-20</div>
                    <div class="flex justify-center">
                      <input
                          type="number"
                          min="0"
                          v-model.number="bluePenalty"
                          class="w-16 rounded-lg bg-white/20 border border-white/30 text-white text-center text-sm font-black focus:outline-none focus:ring-2 focus:ring-white/50 py-1"
                      />
                    </div>
                    <div class="text-sm font-black text-yellow-200 text-center tabular-nums">{{
                        -(bluePenaltyTotal)
                      }}
                    </div>
                  </div>
                </div>

                <!-- Team Names -->
                <div class="mt-4">
                  <div class="text-xs font-black text-white/90 mb-1">Team name</div>
                  <input
                      type="text"
                      v-model="blueTeam1Name"
                      placeholder="Team 1"
                      class="w-full rounded-xl bg-white text-black text-sm font-bold px-3 py-2 mb-2 focus:outline-none focus:ring-2 focus:ring-white/50"
                  />
                  <input
                      type="text"
                      v-model="blueTeam2Name"
                      placeholder="Team 2"
                      class="w-full rounded-xl bg-white text-black text-sm font-bold px-3 py-2 focus:outline-none focus:ring-2 focus:ring-white/50"
                  />
                </div>

                <!-- Total -->
                <div class="mt-4 rounded-2xl bg-blue-600 px-5 py-3 flex items-center justify-between shadow">
                  <div class="text-lg font-black text-white">TOTAL:</div>
                  <div class="text-2xl font-black text-white tabular-nums">{{ blueScore }}</div>
                </div>
              </div>

            </div>

            <!-- ACTIONS -->
            <div class="flex flex-row gap-3 justify-center mt-4">
              <button
                  class="btn btn-neutral rounded-full px-8 shadow"
                  @click="resetScore"
              >
                Reset
              </button>
              <button
                  class="btn btn-success rounded-full px-8 shadow text-white"
                  @click="saveMatch"
              >
                Save
              </button>
              <button
                  class="btn btn-info rounded-full px-8 shadow text-white"
                  @click="showHistory = true"
              >
                History
              </button>
            </div>
          </div>

          <!-- MATCH DATA PAGE -->
          <div class="w-full max-w-4xl px-4" v-else-if="mode === 'MATCH'">
            <div class="flex flex-col items-center gap-6">
              <!-- INPUT FORM CARD -->
              <div class="w-full rounded-3xl bg-base-100 shadow-xl p-5">
                <h2 class="text-lg font-black text-black mb-4">Add Match</h2>

                <!-- Match Number -->
                <div class="mb-4">
                  <div class="text-xs font-black text-black/60 mb-1">Match Number</div>
                  <input
                      type="number"
                      min="1"
                      v-model.number="matchForm.matchNumber"
                      class="w-24 rounded-xl bg-base-200 text-black text-sm font-bold px-3 py-2 focus:outline-none focus:ring-2 focus:ring-error/50"
                  />
                </div>

                <!-- Red vs Blue row -->
                <div class="flex flex-col sm:flex-row gap-4 items-stretch">

                  <!-- Red Alliance Inputs -->
                  <div class="flex-1 rounded-2xl bg-red-500/10 border border-red-500/30 p-4">
                    <div class="text-sm font-black text-red-500 mb-2">RED ALLIANCE</div>
                    <input
                        type="text"
                        v-model="matchForm.redTeam1"
                        placeholder="Team 1"
                        class="w-full rounded-xl bg-white border border-red-200 text-black text-sm font-bold px-3 py-2 mb-2 focus:outline-none focus:ring-2 focus:ring-red-400/50"
                    />
                    <input
                        type="text"
                        v-model="matchForm.redTeam2"
                        placeholder="Team 2"
                        class="w-full rounded-xl bg-white border border-red-200 text-black text-sm font-bold px-3 py-2 focus:outline-none focus:ring-2 focus:ring-red-400/50"
                    />
                  </div>

                  <!-- VS Label -->
                  <div class="flex items-center justify-center">
                    <div
                        class="text-2xl font-black text-transparent bg-clip-text bg-gradient-to-b from-orange-400 to-red-600 drop-shadow-lg"
                        style="text-shadow: 0 2px 8px rgba(0,0,0,0.3);">
                      VS
                    </div>
                  </div>

                  <!-- Blue Alliance Inputs -->
                  <div class="flex-1 rounded-2xl bg-blue-500/10 border border-blue-500/30 p-4">
                    <div class="text-sm font-black text-blue-500 mb-2">BLUE ALLIANCE</div>
                    <input
                        type="text"
                        v-model="matchForm.blueTeam1"
                        placeholder="Team 3"
                        class="w-full rounded-xl bg-white border border-blue-200 text-black text-sm font-bold px-3 py-2 mb-2 focus:outline-none focus:ring-2 focus:ring-blue-400/50"
                    />
                    <input
                        type="text"
                        v-model="matchForm.blueTeam2"
                        placeholder="Team 4"
                        class="w-full rounded-xl bg-white border border-blue-200 text-black text-sm font-bold px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-400/50"
                    />
                  </div>
                </div>

                <!-- Add Button -->
                <div class="mt-4 flex justify-end">
                  <button
                      class="btn btn-success rounded-full px-8 shadow text-white"
                      @click="addMatchEntry"
                  >
                    Add Match
                  </button>
                </div>
              </div>

              <!-- MATCH TABLE -->
              <div class="w-full rounded-3xl bg-base-100 shadow-xl overflow-hidden">
                <div class="px-5 py-4 flex items-center justify-between">
                  <h2 class="text-lg font-black text-black">Match List</h2>
                  <div class="text-xs font-bold text-black/40">{{ matchData.length }}
                    match{{ matchData.length !== 1 ? 'es' : '' }}
                  </div>
                </div>

                <div v-if="matchData.length === 0" class="text-center text-black/40 font-bold py-10">
                  No matches added yet.
                </div>

                <div v-else class="overflow-x-auto">
                  <table class="table w-full">
                    <thead>
                    <tr class="bg-base-200">
                      <th class="text-xs font-black text-black/60 text-left px-5 py-3">#</th>
                      <th class="text-xs font-black text-red-500 text-left px-5 py-3">RED ALLIANCE</th>
                      <th class="text-xs font-black text-black/40 text-center px-2 py-3"></th>
                      <th class="text-xs font-black text-blue-500 text-left px-5 py-3">BLUE ALLIANCE</th>
                      <th class="text-xs font-black text-black/40 text-right px-5 py-3">Actions</th>
                    </tr>
                    </thead>
                    <tbody>
                    <tr
                        v-for="(entry, index) in matchData"
                        :key="index"
                        class="border-t border-base-200"
                    >
                      <!-- Match Number -->
                      <td class="px-5 py-3">
                        <div class="text-sm font-black text-black">{{ entry.matchNumber }}</div>
                      </td>

                      <!-- Red Alliance -->
                      <td class="px-5 py-3">
                        <div class="text-sm font-bold text-black">{{ entry.redTeam.team1Name || '—' }}</div>
                        <div class="text-xs font-bold text-black/40">&amp; {{ entry.redTeam.team2Name || '—' }}</div>
                      </td>

                      <!-- VS -->
                      <td class="text-center px-2 py-3">
                        <div class="text-xs font-black text-black/30">VS</div>
                      </td>

                      <!-- Blue Alliance -->
                      <td class="px-5 py-3">
                        <div class="text-sm font-bold text-black">{{ entry.blueTeam.team1Name || '—' }}</div>
                        <div class="text-xs font-bold text-black/40">&amp; {{ entry.blueTeam.team2Name || '—' }}</div>
                      </td>

                      <!-- Delete -->
                      <td class="px-5 py-3 text-right">
                        <button
                            class="btn btn-ghost btn-xs text-error hover:bg-error/10"
                            @click="confirmDeleteMatch(index)"
                        >
                          Delete
                        </button>
                      </td>
                    </tr>
                    </tbody>
                  </table>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- HISTORY MODAL -->
        <div v-if="showHistory" class="fixed inset-0 z-50 flex items-center justify-center">
          <div class="absolute inset-0 bg-black/40" @click="showHistory = false"></div>
          <div
              class="relative z-10 w-full max-w-2xl max-h-[80vh] overflow-y-auto rounded-3xl bg-base-100 shadow-2xl p-6">

            <div class="flex items-center justify-between mb-4">
              <h2 class="text-2xl font-black tracking-tight text-black ">Match History</h2>
              <button class="btn btn-ghost btn-sm" @click="showHistory = false">✕</button>
            </div>

            <div v-if="history.length === 0" class="text-center text-black/40 font-bold py-12">
              No matches saved yet.
            </div>

            <!-- ENTRIES HISTORY -->
            <div
                v-for="(entry, index) in history"
                :key="index"
                class="rounded-2xl p-5 mb-3 glass shadow-xl"
            >
              <div class="flex items-center justify-between mb-3">
                <div class="text-lg font-black text-black">Match {{ index + 1 }}</div>
                <div class="text-xs font-bold text-black/40">
                  {{ new Date(entry.timestamp).toLocaleDateString() }},
                  {{ new Date(entry.timestamp).toLocaleTimeString() }}
                </div>
              </div>

              <div class="flex items-center justify-around gap-4">
                <!-- Red Side -->
                <div class="flex-1 text-center">
                  <div class="text-sm font-black text-red-500">Red Alliance</div>
                  <div class="text-xs text-black/50 font-bold">
                    {{ entry.redTeam.team1Name || '—' }} & {{ entry.redTeam.team2Name || '—' }}
                  </div>
                  <div class="text-3xl font-black text-red-500 tabular-nums mt-1">{{ entry.redTeam.score }} pts</div>
                </div>

                <!-- Winner Badge -->
                <div class="flex flex-col items-center gap-1">
                  <div class="text-sm font-black text-black/30">VS</div>
                  <div
                      class="rounded-full px-4 py-1 text-xs font-black shadow"
                      :class="entry.redTeam.score > entry.blueTeam.score
                        ? 'bg-red-100 text-red-600'
                        : entry.blueTeam.score > entry.redTeam.score
                          ? 'bg-blue-100 text-blue-600'
                          : 'bg-base-200 text-black/60'"
                  >
                    {{
                      entry.redTeam.score > entry.blueTeam.score
                          ? 'Winner: Red Alliance'
                          : entry.blueTeam.score > entry.redTeam.score
                              ? 'Winner: Blue Alliance'
                              : 'Draw'
                    }}
                  </div>
                </div>

                <!-- Blue Side -->
                <div class="flex-1 text-center">
                  <div class="text-sm font-black text-blue-500">Blue Alliance</div>
                  <div class="text-xs text-black/50 font-bold">
                    {{ entry.blueTeam.team1Name || '—' }} & {{ entry.blueTeam.team2Name || '—' }}
                  </div>
                  <div class="text-3xl font-black text-blue-500 tabular-nums mt-1">{{ entry.blueTeam.score }} pts</div>
                </div>
              </div>
            </div>

          </div>
        </div>

        <!-- DELETE CONFIRMATION MODAL -->
        <div v-if="showDeleteModal" class="fixed inset-0 z-50 flex items-center justify-center">
          <div class="absolute inset-0 bg-black/40" @click="showDeleteModal = false"></div>
          <div class="relative z-10 w-full max-w-sm rounded-3xl bg-base-100 shadow-2xl p-6">
            <div class="flex items-center justify-between mb-4">
              <h2 class="text-lg font-black tracking-tight text-black">Delete Match</h2>
              <button class="btn btn-ghost btn-sm" @click="showDeleteModal = false">✕</button>
            </div>
            <div class="text-sm font-bold text-black/60 mb-6">
              Are you sure you want to delete Match {{
                deleteIndex !== null ? matchData[deleteIndex]?.matchNumber : ''
              }}? This cannot be undone.
            </div>
            <div class="flex justify-end gap-3">
              <button class="btn btn-neutral rounded-full px-6" @click="showDeleteModal = false">Cancel</button>
              <button class="btn btn-error rounded-full px-6 text-white" @click="deleteMatchEntry">Delete</button>
            </div>
          </div>
        </div>

        <!-- STAGE CARDS -->
        <div class="relative z-10 px-6 pb-24" v-if="mode === 'TIMER'">
          <div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
            <div
                class="rounded-3xl p-5 text-white shadow-xl"
                :class="stage > 1 ? 'bg-green-500' : 'bg-red-500'"
            >
              <div class="text-xl font-extrabold drop-shadow">Stage 1</div>
              <div class="mt-1 inline-flex rounded-lg bg-white/90 px-3 py-1 text-xs font-black text-black">
                Automatic Stage
              </div>
              <progress
                  class="progress"
                  :value="stage > 1 ? stageData[1].duration : state === 'RUNNING' && stage === 1 ? stageData[1].duration - timer : 0"
                  :max="stageData[1].duration"
              ></progress>
              <div class="mt-8 text-center text-sm font-bold opacity-95">
                30 seconds
              </div>
            </div>

            <div
                class="rounded-3xl p-5 text-white shadow-xl"
                :class="stage > 2 ? 'bg-green-500' : 'bg-red-500'"
            >
              <div class="text-xl font-extrabold drop-shadow">Stage 2</div>
              <div class="mt-1 inline-flex rounded-lg bg-white/90 px-3 py-1 text-xs font-black text-black">
                Manual Stage
              </div>
              <progress
                  class="progress"
                  :value="stage > 2 ? stageData[2].duration : state === 'RUNNING' && stage === 2 ? stageData[2].duration - timer : 0"
                  :max="stageData[2].duration"
              ></progress>
              <div class="mt-8 text-center text-sm font-bold opacity-95">
                1.40 minutes
              </div>
            </div>

            <div
                class="rounded-3xl p-5 text-white shadow-xl"
                :class="stage > 3 ? 'bg-green-500' : 'bg-red-500'"
            >
              <div class="text-xl font-extrabold drop-shadow">Stage 3</div>
              <div class="mt-1 inline-flex rounded-lg bg-white/90 px-3 py-1 text-xs font-black text-black">
                Modification Stage
              </div>
              <progress
                  class="progress"
                  :value="stage > 3 ? stageData[3].duration : state === 'RUNNING' && stage === 3 ? stageData[3].duration - timer : 0"
                  :max="stageData[3].duration"
              ></progress>
              <div class="mt-8 text-center text-sm font-bold opacity-95">
                60 seconds
              </div>
            </div>

            <div
                class="rounded-3xl p-5 text-white shadow-xl"
                :class="stage > 4 ? 'bg-green-500' : 'bg-red-500'"
            >
              <div class="text-xl font-extrabold drop-shadow">Stage 4</div>
              <div class="mt-1 inline-flex rounded-lg bg-white/90 px-3 py-1 text-xs font-black text-black">
                Final Stage
              </div>
              <progress
                  class="progress"
                  :value="stage > 4 ? stageData[4].duration : state === 'RUNNING' && stage === 4 ? stageData[4].duration - timer : 0"
                  :max="stageData[4].duration"
              ></progress>
              <div class="mt-8 text-center text-sm font-bold opacity-95">
                1.30 minutes
              </div>
            </div>
          </div>
        </div>

        <!-- FOOTER -->
        <div
            class="bg-red-500 py-5 text-center text-white font-extrabold tracking-widest shadow-[0_-8px_30px_rgba(0,0,0,0.18)]">
          ASSUMPTION COLLEGE SAMUTPRAKARN
          <div class="mt-1 text-xs font-black opacity-90">MAKEX D WA</div>
        </div>
      </div>
    </div>

    <!-- MOBILE NAV -->
    <div class="drawer-side">
      <label for="my-drawer-2" aria-label="close sidebar" class="drawer-overlay"></label>
      <ul class="menu bg-base-200 min-h-full w-80 p-4">
        <li class="menu-title"><span>ACSP MakeX Challenge</span></li>
        <li><a @click="mode = 'TIMER'">Timer</a></li>
        <li><a @click="mode = 'SCORE'">Score Sheet</a></li>
        <li><a @click="mode = 'MATCH'">Match Data</a></li>
        <li class="mt-2"><a @click="clearTimer()">Clear Timer</a></li>
      </ul>
    </div>
  </div>
</template>
