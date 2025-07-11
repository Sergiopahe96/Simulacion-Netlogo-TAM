# Simulacion-Netlogo-TAM
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;  Modelo TAM simplificado: Utilidad percibida, Facilidad de uso y Confianza
;  Autores: Sergio Padilla
;  Fecha: Junio 2025
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

; Definición de las tortugas (clientes)
breed [clientes cliente]

; Cada cliente tiene tres atributos clave
clientes-own [
  facilidad-de-uso   ; percepción de la facilidad para usar la IA (0–1)
  utilidad-percibida ; percepción de la utilidad de la IA (0–1)
  confianza          ; nivel de confianza en la IA (0–1)
  riesgo-percibido   ; percepción del riesgo al usar la IA (0–1)
  velocidad          ; velocidad de movimiento
  influencia-social  ; qué tanto se deja influir por otros (0-1)
  adopto-tecnologia  ; booleano: si ya adoptó la tecnología
]

; Variables globales para medir promedios
globals [
  promedio-facilidad
  promedio-utilidad
  promedio-confianza
  promedio-riesgo
 ]

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
; INTERFAZ:
;
; • Slider: num-clientes            (min 10  max 500  step 10)   init 100
; • Slider: peso-uso-facilidad      (min 0   max 1    step 0.05) init 0.6
; • Slider: peso-utilidad-confianza (min 0   max 1    step 0.05) init 0.7
; • Slider: peso-facilidad-confianza(min 0   max 1    step 0.05) init 0.3
; • Slider: ruido-facilidad         (min 0   max 0.5  step 0.01) init 0.1
; • Slider: ruido-utilidad          (min 0   max 0.5  step 0.01) init 0.1
; • Slider: ruido-riesgo           (min 0   max 0.5  step 0.01) init 0.1
; • Slider: peso-riesgo-confianza   (min 0   max 1    step 0.05) init 0.4
; • Slider: peso-riesgo-utilidad    (min 0   max 1    step 0.05) init 0.3
; • Slider: radio-influencia        (min 1   max 10   step 1)    init 3
; • Slider: fuerza-influencia       (min 0   max 1    step 0.05) init 0.2
; • Slider: velocidad-maxima        (min 0.1 max 2    step 0.1)  init 0.5
; • Slider: tendencia-clustering    (min 0   max 1    step 0.05) init 0.3
;
; • Monitor: promedio-facilidad
; • Monitor: promedio-utilidad
; • Monitor: promedio-confianza
; • Monitor: promedio-riesgo
;
; • Plot "Evolución de la confianza":
;    – Serie "Promedio confianza" graficando ticks vs promedio-confianza
;
; • Button "Setup" → setup
; • Button "Go"    → go forever
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

to setup
  clear-all

  ask patches [ set pcolor white ]      ; Fondo blanco
  ; ask patches [ set pcolor black ]    ; Fondo negro
  ; ask patches [ set pcolor gray ]     ; Fondo gris
  ; ask patches [ set pcolor blue ]     ; Fondo azul
  ; ask patches [ set pcolor 8.5 ]      ; Gris claro (usando número)

  ; Crear clientes
  create-clientes num-clientes [
    setxy random-xcor random-ycor

    ; Inicializar percepciones aleatoriamente en [0,1]
    set facilidad-de-uso random-float 1
    set utilidad-percibida random-float 1
    set confianza random-float 1
    set riesgo-percibido random-float 1

    ; Inicializar propiedades de movimiento e interacción
    set velocidad random-float velocidad-maxima
    set influencia-social random-float 1
    set adopto-tecnologia false

    ; Visualización: color y forma basado en nivel de confianza y adopción
    actualizar-apariencia
  ]

  ; Calcular promedios iniciales
  update-globals
  reset-ticks
end

to go
  ask clientes [
    ; Movimiento y posicionamiento
    mover-agente

    ; Interacción social (influencia de vecinos)
    interaccion-social

    ; Actualización de percepciones TAM
    actualizar-facilidad
    actualizar-utilidad
    actualizar-confianza
    actualizar-riesgo

    ; Decidir adopción de tecnología
    decidir-adopcion

    ; Actualizar apariencia visual
    actualizar-apariencia
  ]

  update-globals
  tick
end

; —————————————————————————————————————————————————————————————
;  PROCEDIMIENTOS: MOVIMIENTO E INTERACCIÓN

; Procedimiento de movimiento de agentes
to mover-agente
  ; Movimiento aleatorio con tendencia al clustering
  let target-x xcor
  let target-y ycor

  ; Si hay tendencia a clustering, moverse hacia agentes con alta confianza
  if tendencia-clustering > 0 and any? other clientes in-radius (radio-influencia * 2) [
    let agentes-cercanos other clientes in-radius (radio-influencia * 2)
    let agentes-confiados agentes-cercanos with [confianza > 0.6]

    if any? agentes-confiados [
      let objetivo one-of agentes-confiados
      set target-x [xcor] of objetivo
      set target-y [ycor] of objetivo
    ]
  ]

  ; Movimiento con componente aleatorio y dirigido
  let delta-x (target-x - xcor) * tendencia-clustering + (random-normal 0 1)
  let delta-y (target-y - ycor) * tendencia-clustering + (random-normal 0 1)

  ; Limitar velocidad
  let distancia sqrt (delta-x ^ 2 + delta-y ^ 2)
  if distancia > velocidad [
    set delta-x delta-x * (velocidad / distancia)
    set delta-y delta-y * (velocidad / distancia)
  ]

  ; Mover el agente
  setxy (xcor + delta-x) (ycor + delta-y)
end

; Interacción social - influencia de agentes cercanos
to interaccion-social
  let vecinos other clientes in-radius radio-influencia

  if any? vecinos [
    ; Calcular promedios de vecinos
    let prom-facilidad-vecinos mean [facilidad-de-uso] of vecinos
    let prom-utilidad-vecinos mean [utilidad-percibida] of vecinos
    let prom-confianza-vecinos mean [confianza] of vecinos
    let prom-riesgo-vecinos mean [riesgo-percibido] of vecinos

    ; Influencia basada en la propia susceptibilidad y fuerza global
    let factor-influencia (influencia-social * fuerza-influencia)

    ; Ajustar percepciones hacia el promedio de vecinos
    set facilidad-de-uso facilidad-de-uso +
        (prom-facilidad-vecinos - facilidad-de-uso) * factor-influencia

    set utilidad-percibida utilidad-percibida +
        (prom-utilidad-vecinos - utilidad-percibida) * factor-influencia

    set confianza confianza +
        (prom-confianza-vecinos - confianza) * factor-influencia

    set riesgo-percibido riesgo-percibido +
        (prom-riesgo-vecinos - riesgo-percibido) * factor-influencia

    ; Mantener en rangos válidos
    set facilidad-de-uso max list 0 (min list 1 facilidad-de-uso)
    set utilidad-percibida max list 0 (min list 1 utilidad-percibida)
    set confianza max list 0 (min list 1 confianza)
    set riesgo-percibido max list 0 (min list 1 riesgo-percibido)
  ]
end

; Decisión de adopción de tecnología
to decidir-adopcion
  ; Umbral de adopción considerando riesgo
  let umbral-adopcion (0.6 + (riesgo-percibido * 0.2))  ; más riesgo = umbral más alto

 if not adopto-tecnologia [
    ; Adoptar si hay suficiente confianza, utilidad y poco riesgo
    if (confianza > umbral-adopcion) and (utilidad-percibida > 0.5) and (riesgo-percibido < 0.7) [
      set adopto-tecnologia true
    ]
  ]

  ; Una vez adoptada, es menos probable que la abandone, pero el riesgo puede hacerlo
  if adopto-tecnologia [
    if (confianza < 0.3) or (riesgo-percibido > 0.8) [
      set adopto-tecnologia false
    ]
  ]
end

; Actualizar apariencia visual del agente
to actualizar-apariencia
  ; Color y forma basados en confianza, adopción y riesgo
  ifelse adopto-tecnologia [
    ; Adoptantes: verdes, intensidad por confianza
    set color scale-color green confianza 0 1
    set shape "person"
    set size 1.2
  ] [
    ; No adoptantes: color basado en riesgo vs confianza
    ifelse riesgo-percibido > confianza [
      ; Alto riesgo: tonos naranjas/rojos
      set color scale-color red riesgo-percibido 0 1
    ] [
      ; Baja confianza: tonos azules
      set color scale-color orange confianza 1 0
    ]
    set shape "circle"
    set size 0.8
  ]
end

; —————————————————————————————————————————————————————————————
;  Procedimiento: actualizar-facilidad
;  La percepción de facilidad de uso varía con un ruido controlado
to actualizar-facilidad
  let delta (random-normal 0 ruido-facilidad)
  set facilidad-de-uso facilidad-de-uso + delta

  ; Mantener en rango [0,1]
  if facilidad-de-uso < 0 [ set facilidad-de-uso 0 ]
  if facilidad-de-uso > 1 [ set facilidad-de-uso 1 ]
end

; —————————————————————————————————————————————————————————————
;  Procedimiento: actualizar-utilidad
;  La utilidad percibida se basa en la facilidad y se reduce por el riesgo
to actualizar-utilidad
  let base (peso-uso-facilidad * facilidad-de-uso)
  let penalizacion-riesgo (peso-riesgo-utilidad * riesgo-percibido)
  let ruido random-normal 0 ruido-utilidad

  set utilidad-percibida base - penalizacion-riesgo + ruido

  if utilidad-percibida < 0 [ set utilidad-percibida 0 ]
  if utilidad-percibida > 1 [ set utilidad-percibida 1 ]
end

; —————————————————————————————————————————————————————————————
;  Procedimiento: actualizar-confianza
;  La confianza crece en función de la utilidad y facilidad, se reduce por el riesgo
to actualizar-confianza
  let incremento-utilidad (peso-utilidad-confianza * utilidad-percibida * 0.01)
  let incremento-facilidad (peso-facilidad-confianza * facilidad-de-uso * 0.005)
  let penalizacion-riesgo (peso-riesgo-confianza * riesgo-percibido * 0.01)

  set confianza confianza + incremento-utilidad + incremento-facilidad - penalizacion-riesgo

  if confianza > 1 [ set confianza 1 ]
  if confianza < 0 [ set confianza 0 ]
end

; —————————————————————————————————————————————————————————————
;  Procedimiento: actualizar-riesgo
;  El riesgo percibido puede cambiar por experiencia y influencia social
to actualizar-riesgo
  ; El riesgo tiende a disminuir con la facilidad de uso (familiaridad)
  let reduccion-familiaridad (0.002 * facilidad-de-uso)

  ; Pero puede aumentar por experiencias negativas (ruido)
  let delta (random-normal 0 ruido-riesgo)

  set riesgo-percibido riesgo-percibido - reduccion-familiaridad + delta

  ; Mantener en rango [0,1]
  if riesgo-percibido < 0 [ set riesgo-percibido 0 ]
  if riesgo-percibido > 1 [ set riesgo-percibido 1 ]
end

; —————————————————————————————————————————————————————————————
;  Actualiza las variables globales con los promedios de todos los clientes
to update-globals
  if any? clientes [
    set promedio-facilidad   mean [facilidad-de-uso]   of clientes
    set promedio-utilidad    mean [utilidad-percibida] of clientes
    set promedio-confianza   mean [confianza]          of clientes
    set promedio-riesgo      mean [riesgo-percibido]   of clientes
  ]
end

; —————————————————————————————————————————————————————————————
;  Procedimientos adicionales para análisis

; Reporta el número de clientes con alta confianza (>0.7)
to-report clientes-alta-confianza
  report count clientes with [confianza > 0.7]
end

; Reporta el porcentaje de adopción (clientes con confianza > 0.5)
to-report tasa-adopcion
  if num-clientes > 0 [
    report (count clientes with [adopto-tecnologia] / num-clientes) * 100
  ]
  report 0
end

; Reporta el número de clusters (grupos) formados
to-report numero-clusters
  let clusters 0
  let agentes-sin-contar clientes

  while [any? agentes-sin-contar] [
    let agente-actual one-of agentes-sin-contar
    let grupo agente-actual
    let nuevos-miembros grupo
    let continuar true

    ; Expandir el cluster
    let contador 0
    while [continuar and contador < 10] [
      let candidatos (clientes in-radius-nowrap radio-influencia) with [member? self agentes-sin-contar]
      set nuevos-miembros (turtle-set nuevos-miembros candidatos)
      if count nuevos-miembros = count grupo [
        set continuar false
      ]
      set grupo nuevos-miembros
      set contador contador + 1
    ]

    set agentes-sin-contar agentes-sin-contar with [not member? self grupo]
    set clusters clusters + 1
  ]

  report clusters
end

; Reporta la densidad promedio (agentes por área)
to-report densidad-promedio
  let areas-ocupadas patches with [any? clientes-here]
  if count areas-ocupadas > 0 [
    report count clientes / count areas-ocupadas
  ]
  report 0
end

; Resetea solo las percepciones (mantiene posiciones)
to reset-percepciones
  ask clientes [
    set facilidad-de-uso random-float 1
    set utilidad-percibida random-float 1
    set confianza random-float 1
    set riesgo-percibido random-float 1
    set adopto-tecnologia false
    set color scale-color red confianza 0 1
    actualizar-apariencia
  ]
  update-globals
end
