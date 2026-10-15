<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import bancoInicial from './banco.json'

// Base de datos reactiva
const listaCruda = ref([])

// Estados principales: 'inicio', 'examen', 'resultados'
const pantallaActual = ref('inicio')
const vistaExamen = ref('tarjetas') // 'tarjetas' o 'tabla'

// Configuración del Simulacro (Por defecto 60 preguntas, solo permite 60 o 100)
const cantidadSeleccionada = ref(60)
const macrobloqueSeleccionado = ref('Todos')
const microcontenidoSeleccionado = ref('Todos')
const modoExamen = ref('examen') // 'examen' o 'estudio'
const limiteTiempoMinutos = ref(60)

// Estado del Simulacro activo
const preguntasExamen = ref([])
const indiceActual = ref(0)
const respuestasUsuario = ref({}) // { idxPregunta: 0|1|2|3 }
const marcadasRevision = ref({})
const revelarRespuestaEstudio = ref({})
const filtroRevision = ref('todas')

// Temporizador
const segundosRestantes = ref(0)
const segundosTranscurridos = ref(0)
let intervaloTimer = null

const esLetraRespuesta = (val) => {
  if (val === undefined || val === null) return false
  const limpio = String(val).trim().toUpperCase()
  return ['A', 'B', 'C', 'D'].includes(limpio)
}
const temasVisuales = [
  {
    accent: '#3b82f6',
    hover: '#2563eb',
    soft: 'rgba(59,130,246,.15)',
    g1: '#2563eb',
    g2: '#7c3aed',
    g3: '#06b6d4'
  },

  {
    accent: '#10b981',
    hover: '#059669',
    soft: 'rgba(16,185,129,.15)',
    g1: '#059669',
    g2: '#14b8a6',
    g3: '#84cc16'
  },

  {
    accent: '#ef4444',
    hover: '#dc2626',
    soft: 'rgba(239,68,68,.15)',
    g1: '#dc2626',
    g2: '#f97316',
    g3: '#f59e0b'
  },

  {
    accent: '#8b5cf6',
    hover: '#7c3aed',
    soft: 'rgba(139,92,246,.15)',
    g1: '#7c3aed',
    g2: '#ec4899',
    g3: '#3b82f6'
  },

  {
    accent: '#ec4899',
    hover: '#db2777',
    soft: 'rgba(236,72,153,.15)',
    g1: '#db2777',
    g2: '#9333ea',
    g3: '#6366f1'
  },

  {
    accent: '#f59e0b',
    hover: '#d97706',
    soft: 'rgba(245,158,11,.15)',
    g1: '#d97706',
    g2: '#ef4444',
    g3: '#f97316'
  }
]

const aplicarTemaAleatorio = () => {
  const tema =
    temasVisuales[Math.floor(Math.random() * temasVisuales.length)]

  const root = document.documentElement

  root.style.setProperty('--accent', tema.accent)
  root.style.setProperty('--accent-hover', tema.hover)
  root.style.setProperty('--accent-soft', tema.soft)

  root.style.setProperty('--gradient-1', tema.g1)
  root.style.setProperty('--gradient-2', tema.g2)
  root.style.setProperty('--gradient-3', tema.g3)
}
// Normalizador inteligente: corrige automáticamente si vienen 10 columnas o 12 columnas (con ESTRUCTURAL / INTERMEDIA)
const normalizarPregunta = (item, idx) => {
  // Si el elemento fue guardado con el desfase de 2 columnas (ESTRUCTURAL / INTERMEDIA)
  const valores = [
    item.ID_Pregunta,
    item.Macrobloque,
    item.Microcontenido,
    item.Enunciado,
    item['Opción A'] ?? item['Opcion A'],
    item['Opción B'] ?? item['Opcion B'],
    item['Opción C'] ?? item['Opcion C'],
    item['Opción D'] ?? item['Opcion D'],
    item['Respuesta Correcta'] ?? item['Respuesta correcta'],
    item['Justificación'] ?? item['Justificacion']
  ]

  // Detectar si el Enunciado cayó en "Opción B" (índice 5) por las 2 columnas ocultas (Tipo y Dificultad)
  const tieneDesfaseDosColumnas =
    !esLetraRespuesta(valores[8]) &&
    valores[5] &&
    (String(valores[5]).includes('?') || String(valores[5]).length > 35)

  if (tieneDesfaseDosColumnas) {
    return {
      id: String(valores[0] || `ANAT-${String(idx + 1).padStart(4, '0')}`).trim(),
      tipo: String(valores[1] || '').trim(),
      dificultad: String(valores[2] || '').trim(),
      macrobloque: String(valores[3] || 'General').trim(),
      microcontenido: String(valores[4] || 'General').trim(),
      enunciado: String(valores[5] || '').trim(),
      opciones: [
        String(valores[6] || '').trim(),
        String(valores[7] || '').trim(),
        String(valores[8] || '').trim(),
        String(valores[9] || '').trim()
      ],
      respuestaCorrecta: 0,
      letraCorrecta: 'A',
      justificacion: ''
    }
  }

  // Formato estándar directo desde banco.json (10 o 12 propiedades bien nombradas)
  const id = String(
    item.ID_Pregunta || item.id_pregunta || item.id || `ANAT-${String(idx + 1).padStart(4, '0')}`
  ).trim()
  const tipo = String(item.Tipo || item.tipo || '').trim()
  const dificultad = String(item.Dificultad || item.dificultad || '').trim()
  const macrobloque = String(item.Macrobloque || item.macrobloque || item.tema || 'General').trim()
  const microcontenido = String(item.Microcontenido || item.microcontenido || 'General').trim()
  const enunciado = String(item.Enunciado || item.enunciado || item.pregunta || '').trim()

  const opciones = [
    item['Opción A'] ?? item['Opcion A'] ?? item.opcionA ?? (item.opciones && item.opciones[0]) ?? '',
    item['Opción B'] ?? item['Opcion B'] ?? item.opcionB ?? (item.opciones && item.opciones[1]) ?? '',
    item['Opción C'] ?? item['Opcion C'] ?? item.opcionC ?? (item.opciones && item.opciones[2]) ?? '',
    item['Opción D'] ?? item['Opcion D'] ?? item.opcionD ?? (item.opciones && item.opciones[3]) ?? ''
  ].map((o) => String(o).trim())

  const rawCorrecta = String(
    item['Respuesta Correcta'] ?? item['Respuesta correcta'] ?? item.respuestaCorrecta ?? 'A'
  )
    .trim()
    .toUpperCase()

  const mapaLetras = { A: 0, B: 1, C: 2, D: 3 }
  const indiceCorrecto = mapaLetras[rawCorrecta] !== undefined ? mapaLetras[rawCorrecta] : 0
  const letraCorrecta = ['A', 'B', 'C', 'D'][indiceCorrecto]

  const justificacion = String(
    item['Justificación'] ?? item['Justificacion'] ?? item.justificacion ?? item.explicacion ?? ''
  ).trim()

  return {
    id,
    tipo,
    dificultad,
    macrobloque,
    microcontenido,
    enunciado,
    opciones,
    respuestaCorrecta: indiceCorrecto,
    letraCorrecta,
    justificacion
  }
}

onMounted(() => {
console.log("bancoInicial:", bancoInicial)
console.log("Cantidad:", bancoInicial.length)
  aplicarTemaAleatorio()

  if (Array.isArray(bancoInicial) && bancoInicial.length > 10) {
    listaCruda.value = bancoInicial
    return
  }

  const guardadoLocal = localStorage.getItem('banco_anatomia_custom')
  if (guardadoLocal) {
    try {
      const parsed = JSON.parse(guardadoLocal)
      if (Array.isArray(parsed) && parsed.length > 0) {
        listaCruda.value = parsed
        return
      }
    } catch (e) {
      console.error(e)
    }
  }
  listaCruda.value = bancoInicial
})

const bancoNormalizado = computed(() =>
  listaCruda.value
    .map((item, idx) => normalizarPregunta(item, idx))
    .filter((p) => p.enunciado !== '')
)

// Listas desplegables de Macrobloque y Microcontenido
const macrobloquesDisponibles = computed(() => {
  const set = new Set()
  bancoNormalizado.value.forEach((p) => set.add(p.macrobloque))
  return ['Todos', ...Array.from(set)]
})

const microcontenidosDisponibles = computed(() => {
  const set = new Set()
  bancoNormalizado.value.forEach((p) => {
    if (
      macrobloqueSeleccionado.value === 'Todos' ||
      p.macrobloque === macrobloqueSeleccionado.value
    ) {
      set.add(p.microcontenido)
    }
  })
  return ['Todos', ...Array.from(set)]
})

const preguntasDisponiblesSegunFiltro = computed(() => {
  return bancoNormalizado.value.filter((p) => {
    const pasaMacro =
      macrobloqueSeleccionado.value === 'Todos' || p.macrobloque === macrobloqueSeleccionado.value
    const pasaMicro =
      microcontenidoSeleccionado.value === 'Todos' ||
      p.microcontenido === microcontenidoSeleccionado.value
    return pasaMacro && pasaMicro
  })
})

// Mezclar preguntas (Fisher-Yates)
const mezclarArreglo = (arr) => {
  const copia = [...arr]
  for (let i = copia.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[copia[i], copia[j]] = [copia[j], copia[i]]
  }
  return copia
}

// Generar Simulacro
const generarSimulacro = () => {
  const base = preguntasDisponiblesSegunFiltro.value
  if (base.length === 0) {
    alert('No hay preguntas disponibles con los filtros seleccionados.')
    return
  }

  const mezcladas = mezclarArreglo(base)
  const limite = Math.min(cantidadSeleccionada.value, mezcladas.length)
  preguntasExamen.value = mezcladas.slice(0, limite)

  indiceActual.value = 0
  respuestasUsuario.value = {}
  marcadasRevision.value = {}
  revelarRespuestaEstudio.value = {}
  segundosTranscurridos.value = 0
  segundosRestantes.value = limiteTiempoMinutos.value > 0 ? limiteTiempoMinutos.value * 60 : 0

  pantallaActual.value = 'examen'
  iniciarTemporizador()
}

const borrarRespuestas = () => {
  if (confirm('¿Seguro que deseas dejar vacías todas las respuestas de este simulacro?')) {
    respuestasUsuario.value = {}
    revelarRespuestaEstudio.value = {}
    marcadasRevision.value = {}
  }
}

const iniciarTemporizador = () => {
  detenerTemporizador()
  intervaloTimer = setInterval(() => {
    segundosTranscurridos.value++
    if (limiteTiempoMinutos.value > 0) {
      if (segundosRestantes.value > 0) {
        segundosRestantes.value--
      } else {
        finalizarExamen()
      }
    }
  }, 1000)
}

const detenerTemporizador = () => {
  if (intervaloTimer) {
    clearInterval(intervaloTimer)
    intervaloTimer = null
  }
}

onUnmounted(() => detenerTemporizador())

const preguntaActual = computed(() => preguntasExamen.value[indiceActual.value] || null)
const totalRespondidas = computed(() => Object.keys(respuestasUsuario.value).length)
const porcentajeProgreso = computed(() => {
  if (!preguntasExamen.value.length) return 0
  return Math.round((totalRespondidas.value / preguntasExamen.value.length) * 100)
})

const formatearTiempo = (totalSegundos) => {
  const h = Math.floor(totalSegundos / 3600)
  const m = Math.floor((totalSegundos % 3600) / 60)
  const s = totalSegundos % 60
  const pad = (n) => String(n).padStart(2, '0')
  return h > 0 ? `${pad(h)}:${pad(m)}:${pad(s)}` : `${pad(m)}:${pad(s)}`
}

// Seleccionar o DESELECCIONAR una opción (si vuelve a hacer clic en la misma, queda vacía)
const seleccionarOpcion = (idxPregunta, idxOpcion) => {
  if (respuestasUsuario.value[idxPregunta] === idxOpcion) {
    delete respuestasUsuario.value[idxPregunta]
    delete revelarRespuestaEstudio.value[idxPregunta]
    return
  }

  respuestasUsuario.value[idxPregunta] = idxOpcion
  if (modoExamen.value === 'estudio') {
    revelarRespuestaEstudio.value[idxPregunta] = true
  }
}

const limpiarPreguntaActual = () => {
  delete respuestasUsuario.value[indiceActual.value]
  delete revelarRespuestaEstudio.value[indiceActual.value]
}

const alternarMarca = (idx) => {
  marcadasRevision.value[idx] = !marcadasRevision.value[idx]
}

const irAPregunta = (idx) => {
  if (idx >= 0 && idx < preguntasExamen.value.length) {
    indiceActual.value = idx
  }
}

const confirmarFinalizar = () => {
  const faltantes = preguntasExamen.value.length - totalRespondidas.value
  if (faltantes > 0) {
    if (
      !confirm(
        `Tienes ${faltantes} pregunta(s) en blanco. ¿Deseas finalizar y calificar el simulacro?`
      )
    ) {
      return
    }
  }
  finalizarExamen()
}

const finalizarExamen = () => {
  detenerTemporizador()
  filtroRevision.value = 'todas'
  pantallaActual.value = 'resultados'
}

// Estadísticas idénticas a tu hoja "Resultados" de Excel
const estadisticasResultados = computed(() => {
  let aciertos = 0
  let incorrectas = 0
  let sinResponder = 0

  preguntasExamen.value.forEach((p, idx) => {
    const r = respuestasUsuario.value[idx]
    if (r === undefined) sinResponder++
    else if (r === p.respuestaCorrecta) aciertos++
    else incorrectas++
  })

  const total = preguntasExamen.value.length || 1
  const porcentaje = Number(((aciertos / total) * 100).toFixed(1))
  const notaSobre20 = Number(((aciertos / total) * 20).toFixed(2))

  return { aciertos, incorrectas, sinResponder, total, porcentaje, notaSobre20 }
})

const filasResultados = computed(() => {
  return preguntasExamen.value
    .map((p, idx) => {
      const respIdx = respuestasUsuario.value[idx]
      const letraDada = respIdx !== undefined ? ['A', 'B', 'C', 'D'][respIdx] : 'En blanco'
      const esCorrecta = respIdx === p.respuestaCorrecta
      const noRespondida = respIdx === undefined
      return {
        ...p,
        numero: idx + 1,
        respIdx,
        letraDada,
        esCorrecta,
        noRespondida
      }
    })
    .filter((item) => {
      if (filtroRevision.value === 'correctas') return item.esCorrecta
      if (filtroRevision.value === 'incorrectas') return !item.esCorrecta && !item.noRespondida
      if (filtroRevision.value === 'sinResponder') return item.noRespondida
      return true
    })
})

const letraOpcion = (i) => ['A', 'B', 'C', 'D'][i] || ''
</script>

<template>
  <div class="app-container">
    <!-- BARRA SUPERIOR (SIN BOTÓN DE IMPORTAR) -->
    <header class="navbar">
      <div class="navbar-brand" @click="pantallaActual = 'inicio'" style="cursor: pointer">
        <span class="logo-icon">🧠</span>
        <div>
          <h1>Simulador Automático de Anatomía</h1>
          <span class="subtitle">
            Banco_Preguntas activo: <strong>{{ bancoNormalizado.length }}</strong> preguntas
          </span>
        </div>
      </div>

      <div class="navbar-actions">
        <template v-if="pantallaActual === 'examen'">
          <div class="view-switcher">
            <button
              :class="['switch-btn', { active: vistaExamen === 'tarjetas' }]"
              @click="vistaExamen = 'tarjetas'"
            >
              🗂️ Vista Tarjeta
            </button>
            <button
              :class="['switch-btn', { active: vistaExamen === 'tabla' }]"
              @click="vistaExamen = 'tabla'"
            >
              📊 Vista Hoja Simulacro
            </button>
          </div>

          <div
            class="timer-badge"
            :class="{ 'timer-warning': limiteTiempoMinutos > 0 && segundosRestantes < 300 }"
          >
            <span>⏱️</span>
            <strong v-if="limiteTiempoMinutos > 0">{{ formatearTiempo(segundosRestantes) }}</strong>
            <strong v-else>{{ formatearTiempo(segundosTranscurridos) }}</strong>
          </div>

          <button class="btn-secondary" @click="borrarRespuestas">Borrar Respuestas</button>
          <button class="btn-primary" @click="generarSimulacro">Generar Nuevo</button>
          <button class="btn-success" @click="confirmarFinalizar">Ver Resultados ✓</button>
        </template>
      </div>
    </header>

    <!-- PANTALLA 1: CONFIGURACIÓN DEL SIMULACRO -->
    <main v-if="pantallaActual === 'inicio'" class="main-centered">
      <div class="card setup-card">
        <div class="setup-header">
          <h2>Configuración del Simulacro</h2>
          <p>Selecciona el bloque de estudio y el formato de evaluación.</p>
        </div>

        <div class="setup-grid">
          <!-- Filtro Macrobloque y Microcontenido -->
          <div class="two-cols">
            <div class="form-group">
              <label>Macrobloque</label>
              <select
                v-model="macrobloqueSeleccionado"
                class="select-input"
                @change="microcontenidoSeleccionado = 'Todos'"
              >
                <option v-for="m in macrobloquesDisponibles" :key="m" :value="m">{{ m }}</option>
              </select>
            </div>

            <div class="form-group">
              <label>Microcontenido</label>
              <select v-model="microcontenidoSeleccionado" class="select-input">
                <option v-for="mc in microcontenidosDisponibles" :key="mc" :value="mc">
                  {{ mc }}
                </option>
              </select>
            </div>
          </div>

          <div class="filter-count">
            Preguntas disponibles en este filtro:
            <strong>{{ preguntasDisponiblesSegunFiltro.length }}</strong>
          </div>

          <!-- Cantidad de preguntas: EXCLUSIVAMENTE 60 o 100 -->
          <div class="form-group">
            <label>Cantidad de preguntas del Examen</label>
            <div class="pill-selector">
              <button
                v-for="cant in [60, 100]"
                :key="cant"
                :class="['pill-btn', { active: cantidadSeleccionada === cant }]"
                @click="cantidadSeleccionada = cant"
              >
                {{ cant }} preguntas
              </button>
            </div>
          </div>

          <!-- Modalidad -->
          <div class="form-group">
            <label>Modo de Práctica</label>
            <div class="mode-cards">
              <div
                class="mode-card"
                :class="{ active: modoExamen === 'examen' }"
                @click="modoExamen = 'examen'"
              >
                <div class="mode-title">📝 Modo Simulacro Oficial</div>
                <p>Elige tus respuestas sin ver la solución y obtén tu Nota sobre 20 al final.</p>
              </div>
              <div
                class="mode-card"
                :class="{ active: modoExamen === 'estudio' }"
                @click="modoExamen = 'estudio'"
              >
                <div class="mode-title">💡 Modo Estudio (Con Justificación)</div>
                <p>Muestra al instante la respuesta correcta y la justificación de Moore.</p>
              </div>
            </div>
          </div>

          <!-- Tiempo -->
          <div class="form-group">
            <label>Temporizador</label>
            <div class="pill-selector">
              <button
                v-for="t in [
                  { label: 'Sin límite', val: 0 },
                  { label: '60 min', val: 60 },
                  { label: '90 min', val: 90 },
                  { label: '120 min', val: 120 }
                ]"
                :key="t.val"
                :class="['pill-btn', { active: limiteTiempoMinutos === t.val }]"
                @click="limiteTiempoMinutos = t.val"
              >
                {{ t.label }}
              </button>
            </div>
          </div>
        </div>

        <button class="btn-primary btn-large" @click="generarSimulacro">
          ⚡ Generar Simulacro ({{ cantidadSeleccionada }} preguntas)
        </button>
      </div>
    </main>

    <!-- PANTALLA 2A: EXAMEN EN VISTA DE TARJETAS INTERACTIVAS -->
    <main
      v-else-if="pantallaActual === 'examen' && vistaExamen === 'tarjetas' && preguntaActual"
      class="exam-layout"
    >
      <section class="question-section">
        <div class="progress-wrapper">
          <div class="progress-info">
            <span>Avance del Simulacro</span>
            <span>
              {{ totalRespondidas }} / {{ preguntasExamen.length }} respondidas ({{
                porcentajeProgreso
              }}%)
            </span>
          </div>
          <div class="progress-bar-bg">
            <div class="progress-bar-fill" :style="{ width: porcentajeProgreso + '%' }"></div>
          </div>
        </div>

        <div class="card question-card">
          <div class="question-header">
            <div class="badges-row">
              <span class="badge-id">{{ preguntaActual.id }}</span>
              <span class="badge-macro">{{ preguntaActual.macrobloque }}</span>
              <span class="badge-micro">{{ preguntaActual.microcontenido }}</span>
              <span v-if="preguntaActual.dificultad" class="badge-diff">
                {{ preguntaActual.dificultad }}
              </span>
            </div>

            <div class="question-top-btns">
              <button
                v-if="respuestasUsuario[indiceActual] !== undefined"
                class="btn-clear-ans"
                @click="limpiarPreguntaActual"
              >
                ✕ Dejar en blanco
              </button>
              <button
                class="btn-flag"
                :class="{ flagged: marcadasRevision[indiceActual] }"
                @click="alternarMarca(indiceActual)"
              >
                🚩 {{ marcadasRevision[indiceActual] ? 'Marcada' : 'Marcar' }}
              </button>
            </div>
          </div>

          <div class="question-number">
            Pregunta N.° {{ indiceActual + 1 }} de {{ preguntasExamen.length }}
            <span class="unselect-hint">(Haz clic de nuevo en tu opción para deseleccionarla)</span>
          </div>

          <h2 class="question-text">{{ preguntaActual.enunciado }}</h2>

          <div class="options-list">
            <button
              v-for="(opcion, idxOp) in preguntaActual.opciones"
              :key="idxOp"
              class="option-btn"
              :class="{
                selected: respuestasUsuario[indiceActual] === idxOp,
                correct:
                  modoExamen === 'estudio' &&
                  revelarRespuestaEstudio[indiceActual] &&
                  idxOp === preguntaActual.respuestaCorrecta,
                wrong:
                  modoExamen === 'estudio' &&
                  revelarRespuestaEstudio[indiceActual] &&
                  respuestasUsuario[indiceActual] === idxOp &&
                  idxOp !== preguntaActual.respuestaCorrecta
              }"
              @click="seleccionarOpcion(indiceActual, idxOp)"
            >
              <span class="option-letter">{{ letraOpcion(idxOp) }}</span>
              <span class="option-content">{{ opcion }}</span>
            </button>
          </div>

          <!-- Justificación instantánea en Modo Estudio -->
          <div
            v-if="modoExamen === 'estudio' && revelarRespuestaEstudio[indiceActual]"
            class="feedback-box"
            :class="
              respuestasUsuario[indiceActual] === preguntaActual.respuestaCorrecta
                ? 'feedback-correct'
                : 'feedback-wrong'
            "
          >
            <strong>
              {{
                respuestasUsuario[indiceActual] === preguntaActual.respuestaCorrecta
                  ? '✓ ¡Respuesta Correcta!'
                  : `✕ Incorrecto. La respuesta correcta es la Opción ${preguntaActual.letraCorrecta}.`
              }}
            </strong>
            <p v-if="preguntaActual.justificacion" class="justificacion-text">
              <strong>Justificación:</strong> {{ preguntaActual.justificacion }}
            </p>
          </div>

          <div class="navigation-footer">
            <button
              class="btn-secondary"
              :disabled="indiceActual === 0"
              @click="irAPregunta(indiceActual - 1)"
            >
              ← Anterior
            </button>

            <button
              v-if="indiceActual < preguntasExamen.length - 1"
              class="btn-primary"
              @click="irAPregunta(indiceActual + 1)"
            >
              Siguiente →
            </button>
            <button v-else class="btn-success" @click="confirmarFinalizar">
              Calificar Simulacro ✓
            </button>
          </div>
        </div>
      </section>

      <!-- Navegador de preguntas lateral (optimizado para 60 y 100 preguntas) -->
      <aside class="sidebar-map">
        <div class="card map-card">
          <h3>Navegador de Preguntas</h3>
          <div class="map-legend">
            <span><i class="dot current"></i> Actual</span>
            <span><i class="dot answered"></i> Respondida</span>
            <span><i class="dot flagged"></i> Con bandera</span>
          </div>

          <div class="question-grid">
            <button
              v-for="(preg, idx) in preguntasExamen"
              :key="preg.id"
              class="grid-num-btn"
              :class="{
                current: indiceActual === idx,
                answered: respuestasUsuario[idx] !== undefined,
                flagged: marcadasRevision[idx]
              }"
              @click="irAPregunta(idx)"
            >
              {{ idx + 1 }}
            </button>
          </div>
        </div>
      </aside>
    </main>

    <!-- PANTALLA 2B: EXAMEN EN VISTA DE TABLA (ESTILO HOJA "SIMULACRO" DE EXCEL) -->
    <main
      v-else-if="pantallaActual === 'examen' && vistaExamen === 'tabla'"
      class="table-view-container"
    >
      <div class="card table-card">
        <div class="table-scroll">
          <table class="simulacro-table">
            <thead>
              <tr>
                <th>N</th>
                <th>ID_Pregunta</th>
                <th>Macrobloque</th>
                <th>Microcontenido</th>
                <th class="col-enunciado">Enunciado</th>
                <th>Opción A</th>
                <th>Opción B</th>
                <th>Opción C</th>
                <th>Opción D</th>
                <th>Tu Respuesta</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(preg, idx) in preguntasExamen" :key="preg.id">
                <td class="text-center font-bold">{{ idx + 1 }}</td>
                <td class="text-center nowrap">{{ preg.id }}</td>
                <td class="text-center">{{ preg.macrobloque }}</td>
                <td class="text-center">{{ preg.microcontenido }}</td>
                <td class="col-enunciado">{{ preg.enunciado }}</td>
                <td
                  v-for="(op, idxOp) in preg.opciones"
                  :key="idxOp"
                  class="clickable-option"
                  :class="{ 'cell-selected': respuestasUsuario[idx] === idxOp }"
                  @click="seleccionarOpcion(idx, idxOp)"
                  title="Haz clic para seleccionar o deseleccionar"
                >
                  {{ op }}
                </td>
                <td class="text-center">
                  <select
                    class="table-select"
                    :value="respuestasUsuario[idx] !== undefined ? respuestasUsuario[idx] : ''"
                    @change="
                      (e) =>
                        e.target.value === ''
                          ? delete respuestasUsuario[idx]
                          : (respuestasUsuario[idx] = Number(e.target.value))
                    "
                  >
                    <option value="">En blanco</option>
                    <option :value="0">A</option>
                    <option :value="1">B</option>
                    <option :value="2">C</option>
                    <option :value="3">D</option>
                  </select>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </main>

    <!-- PANTALLA 3: HOJA DE RESULTADOS -->
    <main v-else-if="pantallaActual === 'resultados'" class="results-layout">
      <div class="card results-summary-card">
        <div
          class="score-circle"
          :class="estadisticasResultados.notaSobre20 >= 12.2 ? 'pass' : 'fail'"
        >
          <span class="score-num">{{ estadisticasResultados.notaSobre20 }}</span>
          <span class="score-label">SOBRE 20 PTS</span>
        </div>

        <div class="stats-details">
          <div class="results-header-row">
            <div>
              <h2>Resultados Oficiales del Simulacro</h2>
              <p class="time-spent">
                Tiempo total empleado: {{ formatearTiempo(segundosTranscurridos) }}
              </p>
            </div>
            <div class="results-actions">
              <button class="btn-primary" @click="generarSimulacro">
                🔄 Generar Nuevo Simulacro
              </button>
              <button class="btn-secondary" @click="pantallaActual = 'inicio'">
                ⚙️ Cambiar Filtros
              </button>
            </div>
          </div>

          <div class="stats-boxes">
            <div class="stat-box official">
              <span class="stat-value">
                {{ estadisticasResultados.aciertos }} / {{ estadisticasResultados.total }}
              </span>
              <span class="stat-title">Total Aciertos</span>
            </div>
            <div class="stat-box official-score">
              <span class="stat-value">{{ estadisticasResultados.notaSobre20 }} / 20</span>
              <span class="stat-title">Nota Oficial (sobre 20)</span>
            </div>
            <div class="stat-box correct">
              <span class="stat-value">{{ estadisticasResultados.porcentaje }}%</span>
              <span class="stat-title">Porcentaje</span>
            </div>
            <div class="stat-box wrong">
              <span class="stat-value">{{ estadisticasResultados.sinResponder }}</span>
              <span class="stat-title">En Blanco</span>
            </div>
          </div>
        </div>
      </div>

      <div class="card results-table-card">
        <div class="review-header">
          <h3>Tabla de Calificación y Justificación</h3>
          <div class="pill-selector">
            <button
              :class="['pill-btn', { active: filtroRevision === 'todas' }]"
              @click="filtroRevision = 'todas'"
            >
              Todas ({{ estadisticasResultados.total }})
            </button>
            <button
              :class="['pill-btn', { active: filtroRevision === 'incorrectas' }]"
              @click="filtroRevision = 'incorrectas'"
            >
              Incorrectas ({{ estadisticasResultados.incorrectas }})
            </button>
            <button
              :class="['pill-btn', { active: filtroRevision === 'correctas' }]"
              @click="filtroRevision = 'correctas'"
            >
              Correctas ({{ estadisticasResultados.aciertos }})
            </button>
            <button
              :class="['pill-btn', { active: filtroRevision === 'sinResponder' }]"
              @click="filtroRevision = 'sinResponder'"
            >
              En blanco ({{ estadisticasResultados.sinResponder }})
            </button>
          </div>
        </div>

        <div class="table-scroll">
          <table class="resultados-table">
            <thead>
              <tr>
                <th>No.</th>
                <th>ID_Pregunta</th>
                <th>Tema</th>
                <th>Enunciado y Opciones</th>
                <th>Tu Respuesta</th>
                <th>Respuesta Correcta</th>
                <th>Acierto</th>
                <th>Justificación</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in filasResultados"
                :key="item.id"
                :class="item.esCorrecta ? 'row-ok' : 'row-bad'"
              >
                <td class="text-center font-bold">{{ item.numero }}</td>
                <td class="text-center nowrap font-mono">{{ item.id }}</td>
                <td>
                  <div class="macro-label">{{ item.macrobloque }}</div>
                  <div class="micro-label">{{ item.microcontenido }}</div>
                </td>
                <td>
                  <div class="enunciado-cell">{{ item.enunciado }}</div>
                  <div class="opciones-mini">
                    <span
                      v-for="(op, idxOp) in item.opciones"
                      :key="idxOp"
                      class="mini-opt"
                      :class="{
                        'mini-correct': idxOp === item.respuestaCorrecta,
                        'mini-wrong': idxOp === item.respIdx && !item.esCorrecta
                      }"
                    >
                      <strong>{{ letraOpcion(idxOp) }}:</strong> {{ op }}
                    </span>
                  </div>
                </td>
                <td class="text-center font-bold">{{ item.letraDada }}</td>
                <td class="text-center font-bold text-success">{{ item.letraCorrecta }}</td>
                <td class="text-center">
                  <span v-if="item.esCorrecta" class="badge-acierto ok">✓ 1</span>
                  <span v-else class="badge-acierto bad">✕ 0</span>
                </td>
                <td class="justificacion-cell">{{ item.justificacion }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </main>
  </div>
</template>

<style>
:root {
  --bg-main: #0f172a;
  --bg-card: #1e293b;
  --bg-card-hover: #334155;

  --text-primary: #f8fafc;
  --text-secondary: #94a3b8;

  --accent: #3b82f6;
  --accent-hover: #2563eb;

  --accent-soft: rgba(59,130,246,.15);

  --gradient-1: #3b82f6;
  --gradient-2: #8b5cf6;
  --gradient-3: #06b6d4;

  --purple: #9333ea;
  --success: #10b981;
  --danger: #ef4444;
  --warning: #f59e0b;

  --border: #334155;
}
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: 'Segoe UI', 'Inter', -apple-system, sans-serif;

  background:
    radial-gradient(
      circle at top left,
      var(--gradient-1) 0%,
      transparent 35%
    ),
    radial-gradient(
      circle at top right,
      var(--gradient-2) 0%,
      transparent 35%
    ),
    radial-gradient(
      circle at bottom center,
      var(--gradient-3) 0%,
      transparent 40%
    ),
    var(--bg-main);

  color: var(--text-primary);
  line-height: 1.5;

  background-attachment: fixed;
  min-height: 100vh;
}

.app-container {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* NAVBAR */
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;

  padding: 0.9rem 2rem;

  background:
    linear-gradient(
      135deg,
      rgba(15,23,42,.95),
      rgba(30,41,59,.95)
    );

  border-bottom: 1px solid var(--border);

  position: sticky;
  top: 0;
  z-index: 20;

  backdrop-filter: blur(16px);

  flex-wrap: wrap;
  gap: 1rem;
}

.navbar-brand {
  display: flex;
  align-items: center;
  gap: 0.85rem;
}

.logo-icon {
  font-size: 1.8rem;
}

.navbar-brand h1 {
  font-size: 1.15rem;
  font-weight: 700;

  background:
    linear-gradient(
      90deg,
      var(--gradient-1),
      var(--gradient-2),
      var(--gradient-3)
    );

  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
.subtitle {
  font-size: 0.8rem;
  color: var(--text-secondary);
}

.navbar-actions {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.view-switcher {
  display: flex;
  background: var(--bg-main);
  border: 1px solid var(--border);
  border-radius: 8px;
  overflow: hidden;
}

.switch-btn {
  background: transparent;
  color: var(--text-secondary);
  border: none;
  padding: 0.45rem 0.85rem;
  font-size: 0.85rem;
  cursor: pointer;
  font-weight: 600;
}

.switch-btn.active {
  background: var(--accent);
  color: white;
}

.timer-badge {
  background: var(--bg-main);
  border: 1px solid var(--border);
  padding: 0.4rem 0.85rem;
  border-radius: 8px;
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.timer-warning {
  border-color: var(--danger);
  color: var(--danger);
}

/* CARDS & SETUP */
.card {
  background: rgba(30, 41, 59, 0.75);

  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);

  border: 1px solid rgba(255,255,255,0.08);

  border-radius: 14px;
  padding: 1.5rem;

  box-shadow:
    0 10px 25px rgba(0, 0, 0, 0.35),
    0 0 30px var(--accent-soft);

  transition: all .25s ease;
}
.card:hover {
  transform: translateY(-3px);

  box-shadow:
    0 15px 40px rgba(0,0,0,.4),
    0 0 40px var(--accent-soft);
}
.main-centered {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
}

.setup-card {
  max-width: 720px;
  width: 100%;
}

.setup-header {
  margin-bottom: 1.5rem;
}

.setup-grid {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
  margin-bottom: 1.75rem;
}

.two-cols {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.filter-count {
  font-size: 0.9rem;
  color: #93c5fd;
  background: rgba(59, 130, 246, 0.1);
  padding: 0.5rem 0.85rem;
  border-radius: 8px;
}

.form-group label {
  display: block;
  font-weight: 600;
  margin-bottom: 0.5rem;
  font-size: 0.92rem;
}

.select-input {
  width: 100%;
  padding: 0.65rem;
  border-radius: 8px;
  background: var(--bg-main);
  border: 1px solid var(--border);
  color: var(--text-primary);
  font-size: 0.95rem;
}

.pill-selector {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.pill-btn {
  background: var(--bg-main);
  color: var(--text-primary);
  border: 1px solid var(--border);
  padding: 0.55rem 1.25rem;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 600;
}

.pill-btn.active {
  background: var(--accent);
  border-color: var(--accent);
  color: white;
}

.mode-cards {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.mode-card {
  background: var(--bg-main);
  border: 2px solid var(--border);
  padding: 1rem;
  border-radius: 10px;
  cursor: pointer;
}

.mode-card.active {
  border-color: var(--accent);
  background: rgba(59, 130, 246, 0.1);
}

.mode-title {
  font-weight: 700;
  margin-bottom: 0.3rem;
}

.mode-card p {
  font-size: 0.84rem;
  color: var(--text-secondary);
}

/* BOTONES */
.btn-primary,
.btn-secondary,
.btn-danger,
.btn-success {
  padding: 0.55rem 1.1rem;
  border-radius: 8px;
  font-weight: 600;
  border: none;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.15s;
}

.btn-primary {
  background:
    linear-gradient(
      135deg,
      var(--gradient-1),
      var(--gradient-2)
    );

  color: white;

  box-shadow:
    0 0 20px var(--accent-soft);

  transform: translateY(0);

  transition: all .2s ease;
}
  .btn-primary:hover {
  transform: translateY(-2px);

  box-shadow:
  0 0 40px var(--accent-soft);
}
.btn-secondary {
  background: var(--bg-main);
  color: var(--text-primary);
  border: 1px solid var(--border);
}
.btn-danger {
  background: rgba(239, 68, 68, 0.2);
  color: #fca5a5;
  border: 1px solid var(--danger);
}
.btn-success {
  background: var(--success);
  color: white;
}
.btn-large {
  width: 100%;
  padding: 0.9rem;
  font-size: 1.05rem;
}

/* EXAMEN TARJETAS */
.exam-layout {
  display: grid;
  grid-template-columns: 1fr 330px;
  gap: 1.5rem;
  max-width: 1340px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem 2rem;
}

.progress-wrapper {
  margin-bottom: 1rem;
}
.progress-info {
  display: flex;
  justify-content: space-between;
  font-size: 0.85rem;
  color: var(--text-secondary);
  margin-bottom: 0.35rem;
}

.progress-bar-bg {
  height: 8px;
  background: var(--bg-card);
  border-radius: 99px;
  overflow: hidden;
}

.progress-bar-fill {
  height: 100%;

  background:
    linear-gradient(
      90deg,
      var(--gradient-1),
      var(--gradient-2),
      var(--gradient-3)
    );

  transition: width .3s ease;

  box-shadow:
    0 0 15px var(--gradient-1),
    0 0 25px var(--gradient-2);
}

.question-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.badges-row {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
}

.badge-id {
  background: #334155;
  color: #f8fafc;
  padding: 0.2rem 0.65rem;
  border-radius: 6px;
  font-size: 0.78rem;
  font-family: monospace;
  font-weight: 700;
}

.badge-macro {
  background: rgba(147, 51, 234, 0.2);
  color: #d8b4fe;
  padding: 0.2rem 0.7rem;
  border-radius: 99px;
  font-size: 0.78rem;
  font-weight: 600;
}

.badge-micro {
  background: rgba(59, 130, 246, 0.15);
  color: #93c5fd;
  padding: 0.2rem 0.7rem;
  border-radius: 99px;
  font-size: 0.78rem;
}

.badge-diff {
  background: rgba(245, 158, 11, 0.15);
  color: #fcd34d;
  padding: 0.2rem 0.65rem;
  border-radius: 99px;
  font-size: 0.75rem;
}

.question-top-btns {
  display: flex;
  gap: 0.5rem;
}

.btn-clear-ans {
  background: rgba(239, 68, 68, 0.12);
  border: 1px solid rgba(239, 68, 68, 0.4);
  color: #fca5a5;
  padding: 0.3rem 0.7rem;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.8rem;
}

.btn-flag {
  background: transparent;
  border: 1px solid var(--border);
  color: var(--text-secondary);
  padding: 0.3rem 0.7rem;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.8rem;
}

.btn-flag.flagged {
  border-color: var(--warning);
  color: var(--warning);
}

.question-number {
  color: var(--text-secondary);
  font-size: 0.88rem;
  margin-bottom: 0.4rem;
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;
}

.unselect-hint {
  font-size: 0.78rem;
  color: #64748b;
}

.question-text {
  font-size: 1.2rem;
  margin-bottom: 1.4rem;
  font-weight: 600;
}

.options-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-bottom: 1.4rem;
}

.option-btn {
  display: flex;
  align-items: center;
  gap: 1rem;
  text-align: left;
  width: 100%;
  padding: 0.95rem;
  border-radius: 10px;
  background: var(--bg-main);
  border: 2px solid var(--border);
  color: var(--text-primary);
  cursor: pointer;
  font-size: 0.98rem;
}

.option-btn.selected {
  border-color: var(--accent);

  background: var(--accent-soft);

  box-shadow:
    0 0 20px var(--accent-soft);
}

.option-btn.correct {
  border-color: var(--success);
  background: rgba(16, 185, 129, 0.16);
}

.option-btn.wrong {
  border-color: var(--danger);
  background: rgba(239, 68, 68, 0.16);
}

.option-letter {
  width: 32px;
  height: 32px;
  border-radius: 8px;
  background: var(--bg-card);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  flex-shrink: 0;
}

.feedback-box {
  padding: 1rem;
  border-radius: 10px;
  margin-bottom: 1.4rem;
}

.feedback-correct {
  background: rgba(16, 185, 129, 0.12);
  border: 1px solid var(--success);
}

.feedback-wrong {
  background: rgba(239, 68, 68, 0.12);
  border: 1px solid var(--danger);
}

.justificacion-text {
  margin-top: 0.4rem;
  font-size: 0.92rem;
}

.navigation-footer {
  display: flex;
  justify-content: space-between;
  border-top: 1px solid var(--border);
  padding-top: 1.2rem;
}

/* MAPA */
.map-legend {
  display: flex;
  gap: 0.75rem;
  font-size: 0.78rem;
  color: var(--text-secondary);
  margin: 0.65rem 0 1rem;
}

.dot {
  display: inline-block;
  width: 9px;
  height: 9px;
  border-radius: 50%;
  margin-right: 4px;
}

.dot.current {
  background: var(--accent);
}
.dot.answered {
  background: var(--success);
}
.dot.flagged {
  background: var(--warning);
}

.question-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 0.4rem;
  max-height: 480px;
  overflow-y: auto;
  padding-right: 4px;
}

.grid-num-btn {
  padding: 0.45rem 0;
  border-radius: 6px;
  background: var(--bg-main);
  border: 1px solid var(--border);
  color: var(--text-secondary);
  font-weight: 600;
  cursor: pointer;
  font-size: 0.8rem;
}

.grid-num-btn.answered {
  background: rgba(16, 185, 129, 0.22);
  border-color: var(--success);
  color: white;
}

.grid-num-btn.flagged {
  border-color: var(--warning);
  color: var(--warning);
}

.grid-num-btn.current {
  background: var(--accent);
  border-color: var(--accent);
  color: white;
}

/* VISTA TABLA ESTILO EXCEL SIMULACRO */
.table-view-container {
  padding: 1.5rem;
}

.table-scroll {
  overflow-x: auto;
}

.simulacro-table,
.resultados-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.86rem;
}

.simulacro-table th {
  background: #166534;
  color: white;
  padding: 0.65rem;
  border: 1px solid #14532d;
}

.simulacro-table td {
  padding: 0.65rem;
  border: 1px solid var(--border);
  background: rgba(22, 101, 52, 0.12);
}

.clickable-option {
  cursor: pointer;
  text-align: center;
  transition: background 0.15s;
}

.clickable-option:hover {
  background: rgba(59, 130, 246, 0.2) !important;
}

.cell-selected {
  background: rgba(16, 185, 129, 0.35) !important;
  font-weight: 700;
  border: 2px solid var(--success) !important;
}

.table-select {
  background: var(--bg-main);
  color: white;
  border: 1px solid var(--border);
  padding: 0.35rem 0.5rem;
  border-radius: 6px;
  font-weight: 700;
}

/* RESULTADOS */
.results-layout {
  max-width: 1340px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem 2rem;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.results-summary-card {
  display: flex;
  align-items: center;
  gap: 2rem;
}

.score-circle {
  width: 150px;
  height: 150px;
  border-radius: 50%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border: 8px solid;
  flex-shrink: 0;
}

.score-circle.pass {
  border-color: var(--success);
  color: var(--success);
}
.score-circle.fail {
  border-color: var(--warning);
  color: var(--warning);
}

.score-num {
  font-size: 2.3rem;
  font-weight: 800;
}
.score-label {
  font-size: 0.75rem;
  font-weight: 700;
}

.stats-details {
  flex: 1;
}

.results-header-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  flex-wrap: wrap;
  gap: 1rem;
}

.results-actions {
  display: flex;
  gap: 0.75rem;
}

.stats-boxes {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 1rem;
}

.stat-box {
  background: var(--bg-main);
  padding: 0.9rem;
  border-radius: 10px;
  text-align: center;
  border: 1px solid var(--border);
}

.stat-box.official-score {
  border-color: var(--purple);
  background: rgba(147, 51, 234, 0.12);
}

.stat-value {
  display: block;
  font-size: 1.45rem;
  font-weight: 800;
}

.stat-title {
  font-size: 0.8rem;
  color: var(--text-secondary);
}

.review-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  flex-wrap: wrap;
  gap: 1rem;
}

.resultados-table th {
  background: #1e1b4b;
  color: white;
  padding: 0.7rem;
  border: 1px solid var(--border);
  text-align: left;
}

.resultados-table td {
  padding: 0.75rem;
  border: 1px solid var(--border);
  vertical-align: top;
}

.macro-label {
  font-weight: 700;
  font-size: 0.82rem;
  color: #d8b4fe;
}
.micro-label {
  font-size: 0.76rem;
  color: var(--text-secondary);
}

.enunciado-cell {
  font-weight: 600;
  margin-bottom: 0.45rem;
}

.opciones-mini {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  font-size: 0.78rem;
}

.mini-opt {
  background: var(--bg-main);
  padding: 0.15rem 0.5rem;
  border-radius: 4px;
  border: 1px solid var(--border);
}

.mini-correct {
  border-color: var(--success);
  color: #6ee7b7;
}

.mini-wrong {
  border-color: var(--danger);
  color: #fca5a5;
}

.badge-acierto {
  padding: 0.2rem 0.55rem;
  border-radius: 99px;
  font-weight: 700;
  font-size: 0.78rem;
}

.badge-acierto.ok {
  background: rgba(16, 185, 129, 0.2);
  color: #6ee7b7;
}
.badge-acierto.bad {
  background: rgba(239, 68, 68, 0.2);
  color: #fca5a5;
}

.justificacion-cell {
  color: #cbd5e1;
  font-size: 0.84rem;
}

.text-center {
  text-align: center;
}
.font-bold {
  font-weight: 700;
}
.nowrap {
  white-space: nowrap;
}
.text-success {
  color: #6ee7b7;
}

@media (max-width: 900px) {
  .exam-layout,
  .two-cols,
  .stats-boxes {
    grid-template-columns: 1fr;
  }
  .results-summary-card {
    flex-direction: column;
  }
}
@keyframes glowMove {

  0% {
    filter: hue-rotate(0deg);
  }

  100% {
    filter: hue-rotate(25deg);
  }
}

body {
  animation: glowMove 8s ease-in-out infinite alternate;
}
</style>