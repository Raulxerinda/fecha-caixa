<template>
  <q-page class="fc-page">
    <div class="fc-title">Fecha Caixa</div>

    <div class="fc-scores">
      <div class="fc-score blue" :class="{ active: turn === 1 && !over }">
        Tu: {{ scores[1] }}
      </div>
      <div class="fc-score red" :class="{ active: turn === 2 && !over }">
        {{ vsPC ? 'PC' : 'Jogador 2' }}: {{ scores[2] }}
      </div>
    </div>

    <div class="fc-board">
      <svg :viewBox="`0 0 ${size} ${size}`" class="fc-svg">
        <!-- caixas -->
        <template v-for="(row, r) in boxes" :key="'br' + r">
          <rect
            v-for="(val, c) in row"
            :key="'b' + r + '-' + c"
            :x="P + c * S"
            :y="P + r * S"
            :width="S"
            :height="S"
            :fill="val === 1 ? '#00c2ee' : val === 2 ? '#ff5f57' : 'transparent'"
          />
        </template>

        <!-- linhas horizontais -->
        <template v-for="(row, r) in h" :key="'hr' + r">
          <g
            v-for="(val, c) in row"
            :key="'h' + r + '-' + c"
            class="fc-line"
            @click="play('h', r, c)"
          >
            <line
              :x1="P + c * S" :y1="P + r * S"
              :x2="P + (c + 1) * S" :y2="P + r * S"
              :class="['vis', val ? 'on p' + val : '']"
            />
            <line
              :x1="P + c * S" :y1="P + r * S"
              :x2="P + (c + 1) * S" :y2="P + r * S"
              class="hit"
            />
          </g>
        </template>

        <!-- linhas verticais -->
        <template v-for="(row, r) in v" :key="'vr' + r">
          <g
            v-for="(val, c) in row"
            :key="'v' + r + '-' + c"
            class="fc-line"
            @click="play('v', r, c)"
          >
            <line
              :x1="P + c * S" :y1="P + r * S"
              :x2="P + c * S" :y2="P + (r + 1) * S"
              :class="['vis', val ? 'on p' + val : '']"
            />
            <line
              :x1="P + c * S" :y1="P + r * S"
              :x2="P + c * S" :y2="P + (r + 1) * S"
              class="hit"
            />
          </g>
        </template>

        <!-- pontos -->
        <template v-for="r in N + 1" :key="'dr' + r">
          <circle
            v-for="c in N + 1"
            :key="'d' + r + '-' + c"
            :cx="P + (c - 1) * S"
            :cy="P + (r - 1) * S"
            r="7"
            fill="#111"
          />
        </template>
      </svg>
    </div>

    <div class="fc-msg">{{ message }}</div>

    <div class="fc-actions">
      <q-btn rounded color="dark" label="Novo jogo" @click="reset" />
      <q-btn
        rounded
        outline
        color="white"
        :label="vsPC ? 'Modo: vs Computador' : 'Modo: 2 Jogadores'"
        @click="toggleMode"
      />
    </div>
  </q-page>
</template>

<script setup>
import { ref, reactive, computed } from 'vue'

const N = 5 // 5x5 caixas = 6x6 pontos
const S = 56 // tamanho de cada caixa
const P = 24 // margem
const size = N * S + 2 * P

const mk = (a, b) => Array.from({ length: a }, () => Array(b).fill(0))

const h = ref(mk(N + 1, N))
const v = ref(mk(N, N + 1))
const boxes = ref(mk(N, N))
const turn = ref(1)
const scores = reactive({ 1: 0, 2: 0 })
const vsPC = ref(true)
const over = ref(false)

const message = computed(() => {
  if (over.value) {
    if (scores[1] === scores[2]) return 'Empate! 🤝'
    const w = scores[1] > scores[2] ? 1 : 2
    if (vsPC.value) return w === 1 ? 'Ganhaste! 🎉' : 'O computador ganhou 🤖'
    return `Jogador ${w} ganhou! 🎉`
  }
  if (vsPC.value) return turn.value === 1 ? 'A tua vez' : 'O computador está a pensar...'
  return `Vez do Jogador ${turn.value}`
})

function sides (r, c) {
  return (
    (h.value[r][c] ? 1 : 0) +
    (h.value[r + 1][c] ? 1 : 0) +
    (v.value[r][c] ? 1 : 0) +
    (v.value[r][c + 1] ? 1 : 0)
  )
}

function adj (type, r, c) {
  const list = type === 'h' ? [[r - 1, c], [r, c]] : [[r, c - 1], [r, c]]
  return list.filter(([a, b]) => a >= 0 && a < N && b >= 0 && b < N)
}

function play (type, r, c, byPC = false) {
  if (over.value) return
  if (!byPC && vsPC.value && turn.value === 2) return
  const arr = type === 'h' ? h.value : v.value
  if (arr[r][c]) return

  arr[r][c] = turn.value
  let got = 0
  for (const [a, b] of adj(type, r, c)) {
    if (!boxes.value[a][b] && sides(a, b) === 4) {
      boxes.value[a][b] = turn.value
      got++
    }
  }
  scores[turn.value] += got

  if (scores[1] + scores[2] === N * N) {
    over.value = true
    return
  }
  if (!got) turn.value = turn.value === 1 ? 2 : 1
  if (vsPC.value && turn.value === 2) setTimeout(pcMove, 600)
}

function pcMove () {
  if (over.value) return
  const free = []
  for (let r = 0; r <= N; r++) for (let c = 0; c < N; c++) if (!h.value[r][c]) free.push(['h', r, c])
  for (let r = 0; r < N; r++) for (let c = 0; c <= N; c++) if (!v.value[r][c]) free.push(['v', r, c])

  // 1) fechar caixa se possível
  let pick = free.find(([t, r, c]) => adj(t, r, c).some(([a, b]) => sides(a, b) === 3))
  // 2) senão, jogada segura (não dá a 3ª linha)
  if (!pick) {
    const safe = free.filter(([t, r, c]) => adj(t, r, c).every(([a, b]) => sides(a, b) < 2))
    if (safe.length) pick = safe[Math.floor(Math.random() * safe.length)]
  }
  // 3) senão, a que dá menos caixas
  if (!pick) {
    let best = Infinity
    for (const m of free) {
      const cost = adj(m[0], m[1], m[2]).filter(([a, b]) => sides(a, b) === 2).length
      if (cost < best) { best = cost; pick = m }
    }
  }
  play(pick[0], pick[1], pick[2], true)
}

function reset () {
  h.value = mk(N + 1, N)
  v.value = mk(N, N + 1)
  boxes.value = mk(N, N)
  scores[1] = 0
  scores[2] = 0
  turn.value = 1
  over.value = false
}

function toggleMode () {
  vsPC.value = !vsPC.value
  reset()
}
</script>

<style scoped>
.fc-page {
  background: #ff5f57;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 16px;
  gap: 14px;
}
.fc-title {
  font-size: 30px;
  font-weight: 900;
  color: #fff;
  letter-spacing: 1px;
}
.fc-scores {
  display: flex;
  gap: 12px;
}
.fc-score {
  background: #2b303a;
  border: 4px solid #000;
  border-radius: 30px;
  padding: 4px 18px;
  font-weight: 800;
  font-size: 18px;
  opacity: 0.6;
}
.fc-score.active { opacity: 1; transform: scale(1.08); }
.fc-score.blue { color: #00c2ee; }
.fc-score.red { color: #ff5f57; }
.fc-board {
  background: #ffe8e6;
  border-radius: 24px;
  padding: 8px;
  width: 100%;
  max-width: 420px;
}
.fc-svg { width: 100%; height: auto; display: block; }
.fc-line { cursor: pointer; }
.vis { stroke: rgba(0, 0, 0, 0.12); stroke-width: 9; stroke-linecap: round; }
.vis.on { stroke: #444; }
.hit { stroke: transparent; stroke-width: 30; }
.fc-msg { color: #fff; font-size: 20px; font-weight: 800; min-height: 28px; }
.fc-actions { display: flex; gap: 10px; flex-wrap: wrap; justify-content: center; }
</style>
