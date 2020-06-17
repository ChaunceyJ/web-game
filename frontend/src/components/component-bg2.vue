<template>
<div class="div_background">
    <canvas id='c'></canvas>
</div>
</template>

<script>
export default {
  name: 'component-bg',
  data () {
    return {
      canvas: null,
      ctx: null,
      screenWidth: 0,
      screenHeight: 0,
      matrix: '01'.split(''),
      fontSize: 10,
      drops: [],
      columns: 0,
      timeout: 0
    }
  },
  mounted () {
    this.prepare()
    this.initBackground()
  },
  methods: {
    prepare: function () {
      this.canvas = document.getElementById('c')
      this.canvas.x = 0
      this.canvas.y = 0
      this.ctx = this.canvas.getContext('2d')
      this.screenWidth = this.canvas.width = window.innerWidth
      this.screenHeight = this.canvas.height = window.innerHeight
      this.columns = this.screenWidth / this.fontSize
      for (var x = 0; x < this.columns; x++) {
        this.drops.push(1)
      }
    },
    draw: function () {
      // 黑色背景
      // 将背景黑色设为透明以展示代码
      this.ctx.fillStyle = 'rgba(0, 0, 0, 0.04)'
      this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height)
      this.ctx.fillStyle = '#0F0' // 绿色的文本
      this.ctx.font = this.fontSize + 'px arial'
      for (var i = 0; i < this.drops.length; i++) {
        // 随机字符
        var text = this.matrix[ Math.floor(Math.random() * this.matrix.length) ]
        this.ctx.fillText(text, i * this.fontSize, this.drops[i] * this.fontSize)
        // 字符沿y轴纵向分布
        if (this.drops[i] * this.fontSize > this.canvas.height && Math.random() > 0.975) {
          this.drops[i] = 0
        }
        // 增加y坐标，使其下移
        this.drops[i]++
      }
      if (window.innerWidth !== this.canvas.width) {
        this.onWindowResize()
      }
    },
    onWindowResize: function () {
      this.canvas.height = window.innerHeight
      this.canvas.width = window.innerWidth
      this.columns = this.canvas.width / this.fontSize
      this.drops = []
      for (var x = 0; x < this.columns; x++) {
        this.drops[x] = 1
      }
    },
    initBackground: function () {
      window.addEventListener('resize', this.onWindowResize, false)
      this.timeout = setInterval(() => {
        this.draw()
      }, 35 * 2)
    }
  }
}
</script>

<style scoped>
.div_background {
  width: 100%;
  height: 100%;
  display: flex;
  display: -webkit-flex;
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: -1;
}

.canvas {
  position: absolute;
  left: 0px;
  top: 0px;
}
</style>
