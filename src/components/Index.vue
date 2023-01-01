<template>
  <v-container class="fill-height">
    <v-toolbar
      elevation="5"
      style="padding-bottom: 10px; padding-top: 10px"
>
      <v-btn icon @click="restart">
        <v-icon>fas fa-arrow-rotate-left</v-icon>
      </v-btn>
      <v-btn
        rounded="pill"
        size="large"
        style="background-color: black"
        color="white"
        @click="start"
      >
        <v-icon :icon="playIcon"></v-icon>
      </v-btn>
      <v-btn icon @click="forward">
        <v-icon>fas fa-forward-step</v-icon>
      </v-btn>
      <div class="control" style="max-width: 100px">
        <span class="label">Steps:</span>
        <span class="value" id="iter-number">{{ step }}</span>
      </div>
      <div class="control">
        <span class="label">Prior Mu</span>
        <v-slider
          min="-1.5"
          max="1.5"
          thumb-label="always"
          thumb-size="12"
          step="0.1"
          v-model="pmu"
          hide-details />
      </div>
      <div class="control">
        <span class="label"></span>
        <v-select
          label="Target Function"
          :items="this.targetNames"
          v-model="currentFunc"
          variant="underlined"
          hide-details />
      </div>
      <div class="control">
        <span class="label"></span>
        <v-select
          label="Acquisition Function"
          :items="this.AcqNames"
          v-model="currentAcq"
          variant="underlined"
          hide-details />
      </div>
      <div class="control" v-show="currentAcq === 'UCB'">
        <span class="label">Beta</span>
        <v-slider
          min="0"
          max="100"
          thumb-label="always"
          thumb-size="12"
          step="1"
          v-model="UCBBeta"
          hide-details />
      </div>
      <div class="control" v-show="currentAcq === 'PI'">
        <span class="label">Beta</span>
        <v-slider
          min="-1"
          max="1"
          thumb-label="always"
          thumb-size="12"
          step="0.05"
          v-model="PICThreshold"
          hide-details />
      </div>
      <div class="control" v-show="currentAcq === 'EI'">
        <span class="label">Gamma</span>
        <v-slider
          min="0"
          max="1"
          thumb-label="always"
          thumb-size="12"
          step="0.05"
          v-model="gamma"
          hide-details />
      </div>
    </v-toolbar>
    <v-row class="mt-3" style="height: 85%">
      <div id="graph" />
    </v-row>
  </v-container>
</template>

<script>
import {create, all} from 'mathjs'

const config = { }
const math = create(all, config)
/*
class Matrix {
  constructor(array, T=false) {
    this.m = array.length
    this.n = array[0].length
    this.array = array
    if (T) {
      this.T()
    }
  }

  T() {
    let array = []
    for (let i = 0; i < this.n; i++) {
      array.push([])
      for (let j = 0; j < this.m; j++) {
        array[i][j] = this.array[j][i]
      }
    }
    let t = this.m
    this.m = this.n
    this.n = t
  }

  dot(m) {
    ret = []

}
}*/
const RANGE = [0, 10]

export default {
  data() {
    return {
      play: false,
      step: 0,
      x: [],
      range: RANGE,
      divide: 200,
      history: [[], []],
      targets: {
        Sin(x) {
          return Math.sin(x*2);
        },
        Sigmoid(x) {
          return (1 / (1 + 2.718 ** (-x+(RANGE[0]+RANGE[1])/2)))
        },
        Square(x) {
          return (x - (RANGE[0]+RANGE[1])/2) ** 2 / 10
        },
        XSinXRec(x) {
          x = (x - (RANGE[0]+RANGE[1])/2)*0.1
          if (x === 0) return 0
          return 2 * x * math.sin((1/x))
        },
        X2SinX(x) {
          return Math.sin((x-4.5)*2) * ((x - (RANGE[0]+RANGE[1])/2) ** 2 / 10 -2 )
        },
        heart(x) {
          x = (x - (RANGE[0]+RANGE[1])/2) / 2
          return ((x**2)**(1/3) + 0.9 * math.sqrt(8 - x ** 2) * math.sin(8*3.14159*x)) / 3
        }
      },
      acquisitions: {
        history: [[], []],
        x: [],
        Random(x) {
          let ret = []
          for (let i = 0; i < x.length; i++) {
            ret.push(math.random(0, 1))
          }
          return ret
        },
        UCB(mu, cov, beta) {
          return math.subtract(math.multiply(beta, math.diag(cov)), mu).toArray()
        },
        PI(mu, cov, b, c) {
          let ret = []
          cov = math.diag(cov).toArray()
          for (let i = 0; i < mu.length; i++) {
            let y = 0.5*(1+math.sqrt(1-math.exp(-math.sqrt(math.pi/8)*(c-mu[i])**2/cov[i])))
            if (c - mu[i] < 0) y = 1 - y
            ret.push(y)
          }
          return ret
        },
        EI(x, cov, b, c, gamma) {
          if (this.history[0].length < 1) {
            return this.Random(x)
          }
          let h1 = [] // lower
          let h2 = [] // greater
          let mid = math.quantileSeq(this.history[1], gamma)
          for (let i = 0; i < this.history[0].length; i++) {
            if (this.history[1][i] <= mid) {
              h1.push(this.history[0][i])
            } else {
              h2.push(this.history[0][i])
            }
          }
          if (h1.length < 2 || h2.length < 2) {
            return this.Random(x)
          }
          let ret = []
          let mu1 = math.sum(h1) / h1.length
          let mu2 = math.sum(h2) / h2.length
          let si1 = math.variance(h1)+1e-8
          let si2 = math.variance(h2)+1e-8
          let gauss = function (x, mu, si) {
            return 1/(math.sqrt(si*2*math.pi))*math.exp(0-(x-mu)**2/(2*si))
          }
          for (let xi of this.x) {
            ret.push(1/(gamma+gauss(xi, mu2, si2)/gauss(xi, mu1, si1)*(1-gamma)))
          }
          return ret
        }
      },
      currentFunc: 'Sin',
      currentAcq: 'UCB',
      nextChoice: 0,
      pmu: 0,
      UCBBeta: 30,
      curBest: 0,
      PICThreshold: -0.1,
      gamma: 0.25,
      repeat: undefined,
    }
  },
  methods: {
    restart() {
      this.history = [[], []]
      this.acquisitions.history = this.history
      this.draw()
      this.step = 0
      this.play = false
      this.curBest = 0
      window.clearInterval(this.repeat)
    },
    start() {
      if (!this.play) {
        this.forward()
        this.repeat = setInterval(this.forward, 700)
      } else {
        window.clearInterval(this.repeat)
      }
      this.play = !this.play
    },
    forward() {
      this.explore(this.nextChoice)
      this.step++
    },
    draw() {
      let mu, cov
      if (this.history[0].length) {
        [mu, cov] = this.update(this.history[0], this.history[1], this.x, this.pmu)
        mu = mu.toArray()
      } else {
        mu = []
        for (let i = 0; i < this.x.length; i++) {
          mu.push(this.pmu)
        }
        cov = this.gaussianK(this.x, this.x)
      }
      let bound = math.multiply(1.96, math.diag(cov).map(math.sqrt))
      let traceMu = {
        x: this.x,
        y: mu,
        line: {color: "rgb(87,0,222)"},
        mode: "lines",
        name: "Mu",
        type: "scatter"
      }
      let traceReal = {
        x: this.x,
        y: this.x.map(this.targets[this.currentFunc]),
        line: {color: "rgb(0,0,0)"},
        mode: "lines",
        name: "Real",
        type: "scatter"
      }
      let dots = {
        x: this.history[0],
        y: this.history[1],
        mode: "markers",
        type: "scatter",
        color: "rgb(255,0,0)",
        name: "data points"
      }
      let traceVar = {
        x: this.x.concat(this.x.slice().reverse()),
        y: math.add(mu, bound).toArray().concat(math.subtract(mu, bound).toArray().reverse()),
        fill: "tozerox",
        fillcolor: "rgba(103,157,255,0.2)",
        line: {color: "transparent"},
        name: "95% Confidence Bound",
        type: "scatter"
      }
      let acqY = this.acquisitions[this.currentAcq](mu, cov, this.UCBBeta, this.curBest + this.PICThreshold, this.gamma)
      let t = math.randomInt(0, this.x.length-1)
      let bigY = acqY[t]
      let bigX = this.x[t]
      for (let i = 0; i < this.x.length; i++) {
        if (acqY[i] > bigY) {
          bigY = acqY[i]
          bigX = this.x[i]
        }
      }
      this.nextChoice = bigX
      let dotNext = {
        x: [bigX],
        y: [bigY],
        mode: 'markers',
        type: 'scatter',
        name: 'Next Choice',
        marker: {size: 8, color: 'rgb(255,222,0)'},
        yaxis: 'y2'
      }
      let traceAcq = {
        x: this.x,
        y: acqY,
        yaxis: 'y2'
      }
      let data = [traceMu, traceReal, traceVar, dots, traceAcq, dotNext];
      let layout = {
        paper_bgcolor: "rgb(255,255,255)",
        plot_bgcolor: "rgb(229,229,229)",
        xaxis: {
          gridcolor: "rgb(255,255,255)",
          range: this.range,
          showgrid: true,
          showline: false,
          showticklabels: true,
          tickcolor: "rgb(127,127,127)",
          ticks: "outside",
          zeroline: false
        },
        yaxis: {
          gridcolor: "rgb(255,255,255)",
          showgrid: true,
          showline: false,
          showticklabels: true,
          tickcolor: "rgb(127,127,127)",
          ticks: "outside",
          zeroline: false,
          domain: [0.31,1],
        },
        yaxis2: {
          gridcolor: "rgb(255,255,255)",
          showgrid: true,
          showline: false,
          showticklabels: true,
          tickcolor: "rgb(127,127,127)",
          ticks: "outside",
          zeroline: false,
          domain: [0,0.29],
        },
        grid: {
          rows: 2,
          columns: 1,
          subplots: [['xy'], ['xy2']],
          roworder: 'top to bottom'
        }
      };
      Plotly.newPlot('graph', data, layout, {displayModeBar: false, responsive: true});
    },
    explore(x) {
      let y = this.targets[this.currentFunc](x)
      if (y < this.curBest) this.curBest = y
      this.history[0].push(x)
      this.history[1].push(y)
      this.draw()
    },
    gaussianK(x1, x2, l=0.5, sigma=0.2) {
      let m = math.size(x1)[0]
      let n = math.size(x2)[0]
      let dist = math.zeros([m, n])
      for (let i = 0; i < m; i++) {
        for (let j = 0; j < n; j++) {
          dist[i][j] = math.sum(math.square(x1[i] - x2[j]))
        }
      }
      dist = math.matrix(dist)
      dist = dist.map(function (value) {
        return sigma ** 2 * math.exp(-0.5 / l ** 2 * value)
      })
      dist = math.add(dist, math.multiply(math.identity(m, n), 1e-8)) // avoid O matrix
      return dist
    },
    update(ox, oy, x, pmu) {
      let kxx = this.gaussianK(ox, ox)
      let kss = this.gaussianK(x, x)
      let kxs = this.gaussianK(ox, x)
      let ksx = math.transpose(kxs)
      let kxx_inv = math.inv(kxx)
      let mu = math.chain(ksx).multiply(kxx_inv).multiply(math.subtract(oy, pmu)).add(pmu).done()
      let cov = math.subtract(kss, math.chain(ksx).multiply(kxx_inv).multiply(kxs).done())
      return [mu, cov]
    },
  },
  computed: {
    playIcon() {
      if (!this.play) {
        return "fas fa-play"
      } else {
        return "fas fa-pause"
      }
    },
    targetNames() {
      let ret = []
      for (let f in this.targets) {
        ret.push(f)
      }
      return ret
    },
    AcqNames() {
      let ret = []
      for (let f in this.acquisitions) {
        if (f.charCodeAt(0) > 96) {
          continue
        }
        ret.push(f)
      }
      return ret
    },
  },
  watch: {
    currentFunc() {
      this.restart()
    },
    currentAcq() {
      this.draw()
    },
    pmu() {
      this.draw()
    },
    UCBBeta() {
      this.draw()
    },
    PICThreshold() {
      this.draw()
    },
    gamma() {
      this.draw()
    }
  },
  mounted() {
    let l = this.range[0];
    let r = this.range[1];
    this.x[0] = l;
    let drop = (r - l) / this.divide
    for (let i = l + drop; i <= r; i+=drop) {
      this.x.push(i);
    }
    this.acquisitions.x = this.x
    this.acquisitions.history = this.history
    this.draw()
  }
}
</script>

<style scoped>
.control {
  flex-grow: 1;
  max-width: 150px;
  min-width: 50px;
  margin-left: 20px;
  font-family: "Helvetica", "Arial", sans-serif;
}
.control .label {
  color: #777;
  font-size: 13px;
  display: block;
  font-weight: 300;
  line-height: 20px;
}
.control .value {
  font-size: 24px;
  margin: 0;
  font-weight: 300;
}
#graph {
  width: 100%;
  height: 100%;
}
</style>
