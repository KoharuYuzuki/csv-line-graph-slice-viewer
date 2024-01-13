<script lang="ts">
  export default {
    data(): {
      data: {
        [key: string]: {
          name: string
          data: number[][]
          valueMax: number
          valueMin: number
        }
      }
      canvas: HTMLCanvasElement | null
      canvasWidth: number
      canvasHeight: number
      time: number
      timeAxisLen: number
      dataAxisLen: number
      dataValueMax: number
      dataValueMin: number
    } {
      return {
        data: {},
        canvas: null,
        canvasWidth: 1200,
        canvasHeight: 500,
        time: 0,
        timeAxisLen: 0,
        dataAxisLen: 0,
        dataValueMax: 0,
        dataValueMin: 0
      }
    },
    methods: {
      loadFile() {
        const input = document.createElement('input')
        input.type = 'file'
        input.accept = '.csv'

        const reader = new FileReader()
        let fileName = '名称未設定'

        reader.addEventListener('load', () => {
          const csv = reader.result as string
          const array2d = this.parseCSV(csv)
          const key = crypto.randomUUID()

          const max = array2d.reduce((a, b) => {
            return Math.max(
              a,
              b.reduce((c, d) => Math.max(c, d))
            )
          }, 0)

          const min = array2d.reduce((a, b) => {
            return Math.min(
              a,
              b.reduce((c, d) => Math.min(c, d))
            )
          }, 0)

          this.data[key] = {
            name: fileName,
            data: array2d,
            valueMax: max,
            valueMin: min
          }

          this.updateDataInfo()
          this.draw()
        }, {once: true})

        input.addEventListener('change', () => {
          if (!input.files || (input.files.length <= 0)) return
          const file = input.files[0]
          fileName = file.name
          reader.readAsText(file)
        }, {once: true})

        input.click()
      },
      parseCSV(csv: string) {
        const lines = csv.split('\n')
        const array2d =
          lines
          .filter((line) => line.length > 0)
          .map((line) => line.split(',').map((str) => Number(str)))

        const xAxisLenMax = array2d.reduce((a, b) => Math.max(a, b.length), 0)
        const yAxisLen    = array2d.length

        const rotated = [...Array(xAxisLenMax)].map((_, i) => {
          return [...Array(yAxisLen)].map((_, j) => {
            const value = array2d[j][i]
            return value ? value : 0
          })
        })

        return rotated
      },
      computeTimeAxisLenMax() {
        this.timeAxisLen =
          Object.keys(this.data)
          .map((key) => {
            const data = this.data[key].data
            return data.length
          })
          .reduce((a, b) => Math.max(a, b), 0)
      },
      computeDataAxisLenMax() {
        this.dataAxisLen =
          Object.keys(this.data)
          .map((key) => {
            const data = this.data[key].data
            return data.reduce((a, b) => Math.max(a, b.length), 0)
          })
          .reduce((a, b) => Math.max(a, b), 0)
      },
      computeDataValueMax() {
        this.dataValueMax =
          Object.keys(this.data)
          .map((key) => this.data[key].valueMax)
          .reduce((a, b) => Math.max(a, b), 0)
      },
      computeDataValueMin() {
        this.dataValueMin =
          Object.keys(this.data)
          .map((key) => this.data[key].valueMin)
          .reduce((a, b) => Math.min(a, b), 0)
      },
      draw() {
        if (!this.canvas) return

        const dataAxisLen = this.dataAxisLen

        let valueMax = this.dataValueMax
        let valueMin = this.dataValueMin
        let valueDiff = valueMax - valueMin

        if ((valueMax === 0) && (valueMin === 0)) {
          valueMax = 1
          valueMin = 1
          valueDiff = 2
        }

        const ctx = this.canvas.getContext('2d')
        if (!ctx) return

        const w = this.canvas.width
        const h = this.canvas.height
        const wStep = w / (dataAxisLen - 1)
        const hStep = h / valueDiff
        const hCenter = h / valueDiff * valueMax

        const time = this.time

        const colors = [
          '#F26651',
          '#EFC251',
          '#9DEF51',
          '#51BDEF',
          '#8351EF'
        ]

        ctx.clearRect(0, 0, w, h)
        ctx.beginPath()
        ctx.setLineDash([2, 2])
        ctx.moveTo(0, hCenter)
        ctx.lineTo(w, hCenter)
        ctx.strokeStyle = 'black'
        ctx.stroke()
        ctx.setLineDash([])

        Object.keys(this.data).forEach((key, i) => {
          const data = this.data[key].data[time]
          if (!data || (data.length <= 0)) return

          ctx.beginPath()
          for (let j = 0; j < data.length; j++) {
            const x = j * wStep
            const y = hCenter - (data[j] * hStep)
            ctx.lineTo(x, y)
            ctx.moveTo(x, y)
          }
          ctx.strokeStyle = colors[i] ? colors[i] : 'black'
          ctx.stroke()
        })
      },
      onClick(key: string) {
        delete this.data[key]
        this.updateDataInfo()
        this.draw()
      },
      updateDataInfo() {
        this.computeTimeAxisLenMax()
        this.computeDataAxisLenMax()
        this.computeDataValueMax()
        this.computeDataValueMin()
      }
    },
    watch: {
      time() {
        this.draw()
      }
    },
    mounted() {
      this.canvas = this.$refs.canvas as HTMLCanvasElement
      this.draw()
    }
  }
</script>

<template>
  <button
    v-on:click="loadFile"
  >
    CSVファイルを開く
  </button>
  <canvas
    ref="canvas"
    v-bind:width="canvasWidth"
    v-bind:height="canvasHeight"
  >
  </canvas>
  <input
    type="range"
    min="0"
    v-bind:max="timeAxisLen"
    step="1"
    value="0"
    v-on:input="time = $event.target ? Number(($event.target as HTMLInputElement).value) : 0"
  >
  <p>
    Time: {{ time + 1 }}, Total: {{ timeAxisLen }}
  </p>
  <button
    v-for="key in Object.keys(data)"
    v-bind:key="key"
    v-on:click="onClick(key)"
  >
    データ "{{ data[key].name }}" を削除
  </button>
  <img src="/csv-format.png" width="800">
  <a href="/third-party-licenses.txt">
    サードパーティーライセンス
  </a>
</template>

<style>
  html,
  body {
    width: 100dvw;
    height: 100dvh;
    margin: 0;
    display: flex;
  }

  #app {
    width: calc(100% - 32px);
    height: fit-content;
    min-height: calc(100% - 32px);
    margin: 16px;
    display: flex;
    flex-direction: column;
  }

  button {
    margin-bottom: 16px;
  }

  canvas {
    margin-bottom: 16px;
    border: 2px solid;
  }

  input[type='range'] {
    margin: 0;
    margin-bottom: 16px;
  }

  p {
    margin: 0;
    margin-bottom: 16px;
  }

  a {
    margin-bottom: 16px;
  }
</style>
