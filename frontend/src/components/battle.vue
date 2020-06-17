<template>
  <div>
    <canvas id="canvas_game">
      <img class="img_src" id="img_player1" src="../assets/player1.svg" />
      <img class="img_src" id="img_player2" src="../assets/player2.png"/>
      <img class="img_src" id="img_explosion" src="../assets/explosion.svg" />
    </canvas>

    <div class="div_progress">
      <Progress class="progress_hp" :percent="binderHp2" :stroke-width="10" />
      <Progress class="progress_hp" :percent="binderHp1" :stroke-width="10" />
    </div>

    <comMenu></comMenu>
    <comAlert v-show="alertPause"></comAlert>

    <Modal :closable="false" :mask-closable="false" v-model="modalRecover">
      <h1 slot="header" class="h1_modal">继续上次游戏</h1>
      <div>
        <p class="p_modal">监测到有您上次保存的游戏进度</p>
        <p class="p_modal">是否继续上次游戏？</p>
      </div>
      <div slot="footer">
        <Button type="error" size="large" @click="onRecover(false)">否</Button>
        <Button type="success" size="large" @click="onRecover(true)">是</Button>
      </div>
    </Modal>

    <Modal :closable="false" :mask-closable="false" v-model="modalHelp">
      <h1 slot="header" class="h1_modal">帮助</h1>
      <div>
        <p class="p_modal">1. 玩家相互攻击；</p>
        <p class="p_modal">2. 当任一玩家血条为0时游戏结束；</p>
        <p class="p_modal">3. 点击 空格 可以暂停或继续游戏。</p>
        <Table border :columns="colController" :data="dataController"></Table>
      </div>
      <div slot="footer">
        <Button type="primary" size="large" @click="onStart">我明白了</Button>
      </div>
    </Modal>

    <Modal :closable="false" :mask-closable="false" v-model="modalResult">
      <h1 slot="header" class="h1_modal">游戏结束</h1>
      <div>
        <p class="p_modal">玩家 {{playerWin}} 获胜 !</p>
        <p class="p_modal">想要开启一场新游戏吗？</p>
      </div>
      <div slot="footer">
        <Button type="error" size="large" @click="onResult(false)">否</Button>
        <Button type="success" size="large" @click="onResult(true)">是</Button>
      </div>
    </Modal>

    <Modal :closable="false" :mask-closable="false" v-model="modalSave">
      <h1 slot="header" class="h1_modal">保存游戏进度</h1>
      <div>
        <p class="p_modal">是否保存并退出游戏？</p>
      </div>
      <div slot="footer">
        <Button type="error" size="large" @click="onSave(false)">否</Button>
        <Button type="success" size="large" @click="onSave(true)">是</Button>
      </div>
    </Modal>

  </div>
</template>

<script>
import comMenu from './component-menu.vue'
import comAlert from './component-alert.vue'
import mPlayer from '../res/Player'
import mBullet from '../res/Bullet'
import mGame from '../res/Game'

export default {
  name: 'battle',
  components: {
    comMenu,
    comAlert
  },
  data () {
    return {
      /** Env */
      canvas: null,
      ctx: null,
      screenWidth: 0,
      screenHeight: 0,
      /** Binder */
      binderHp1: 0,
      binderHp2: 0,
      /** Img */
      imgPlayer1: null,
      imgPlayer2: null,
      imgExplosion: null,
      /** Src */
      player1: null,
      player2: null,
      bullets1: [],
      bullets2: [],
      /** Flag */
      isShotting1: false,
      isShotting2: false,
      alertPause: false,
      modalHelp: false,
      modalResult: false,
      modalRecover: false,
      modalSave: false,
      gameState: -1,
      playerWin: -1,
      /** Handler */
      animationIdLoop: 0,
      timeoutIdStop: 0,
      /** Table */
      colController: [
        {
          title: '移动方向',
          key: 'operation'
        },
        {
          title: '玩家1',
          key: 'player1'
        },
        {
          title: '玩家2',
          key: 'player2'
        }
      ],
      dataController: [
        {
          operation: '向上移动',
          player1: 'W',
          player2: '⬆️'
        },
        {
          operation: '向下移动',
          player1: 'S',
          player2: '⬇️'
        },
        {
          operation: '向左移动',
          player1: 'A',
          player2: '⬅️'
        },
        {
          operation: '向右移动',
          player1: 'D',
          player2: '➡️'
        },
        {
          operation: '射击',
          player1: 'F',
          player2: 'L'
        }
      ]
    }
  },
  mounted () {
    if (typeof localStorage.battle !== 'undefined') {
      this.modalRecover = true
    } else {
      this.onPrepare()
    }
  },
  methods: {
    onRecover: function (n) {
      this.modalRecover = false
      if (n) {
        var lastInfo = JSON.parse(localStorage.battle)
        // console.log(lastInfo)
        localStorage.removeItem('battle')
        if (this.gameState === mGame.GameState.CREATE) {
          this.prepareEnv()
          this.prepareImg()
          this.recoverSrc(lastInfo)
          this.attachListener()
        } else {
          this.prepareEnv()
          this.recoverSrc(lastInfo)
        }
        this.gameState = mGame.GameState.PREPARE
        this.onStart()
      } else {
        localStorage.removeItem('battle')
        this.onPrepare()
      }
    },
    recoverSrc: function (data) {
      this.alertPause = data.alertPause
      this.animationIdLoop = data.animationIdLoop
      this.timeoutIdStop = data.timeoutIdStop
      this.player1 = new mPlayer.Player(
        this.canvas,
        data.player1.x,
        data.player1.y,
        this.imgPlayer1,
        this.imgExplosion,
        null,
        null
      )
      this.player2 = new mPlayer.Player(
        this.canvas,
        data.player2.x,
        data.player2.y,
        this.imgPlayer2,
        this.imgExplosion,
        null,
        null
      )
      this.binderHp1 = data.player1.hp
      this.binderHp2 = data.player2.hp
      this.player1.setState(data.player1.hp, data.player1.alive, data.player1.laser, data.player1.shield)
      this.player2.setState(data.player2.hp, data.player2.alive, data.player2.laser, data.player2.shield)
      data.bullets1.forEach(element => {
        var bt = this.createPlayerBullet(element.x, element.y, element.offsetY, element.color)
        this.bullets1.push(bt)
      })
      data.bullets2.forEach(element => {
        var bt = this.createPlayerBullet(element.x, element.y, element.offsetY, element.color)
        this.bullets2.push(bt)
      })
    },
    /** Lifecycle */
    onPrepare: function () {
      if (this.gameState === mGame.GameState.CREATE) {
        this.prepareEnv()
        this.prepareImg()
        this.prepareSrc()
        this.attachListener()
        this.modalHelp = true
      } else {
        this.prepareEnv()
        this.prepareSrc()
      }
      this.gameState = mGame.GameState.PREPARE
    },
    onStart: function () {
      this.modalHelp = false
      if (this.gameState === mGame.GameState.PREPARE) {
        this.gameState = mGame.GameState.START
        this.onResume()
      }
    },
    onResume: function () {
      this.alertPause = false
      this.gameState = mGame.GameState.RUN
      this.loop()
    },
    onPause: function () {
      this.isShotting1 = false
      this.isShotting2 = false
      this.player1.resetStates()
      this.player2.resetStates()
      this.alertPause = true
      this.gameState = mGame.GameState.PAUSE
    },
    onStop: function () {
      this.isShotting1 = false
      this.isShotting2 = false
      this.player1.resetStates()
      this.player2.resetStates()
      this.timeoutIdStop = setTimeout(() => {
        this.clearGameTasks()
        this.gameState = mGame.GameState.STOP
        this.modalResult = true
      }, 2000)
    },
    /** Prepare */
    prepareEnv: function () {
      this.canvas = document.getElementById('canvas_game')
      this.ctx = this.canvas.getContext('2d')
      this.screenWidth = this.canvas.width = window.innerWidth
      this.screenHeight = this.canvas.height = window.innerHeight
    },
    prepareImg: function () {
      this.imgPlayer1 = document.getElementById('img_player1')
      this.imgPlayer2 = document.getElementById('img_player2')
      this.imgExplosion = document.getElementById('img_explosion')
    },
    prepareSrc: function () {
      this.player1 = this.createPlayer(this.screenHeight - this.imgPlayer1.height / 2, this.imgPlayer1)
      this.player2 = this.createPlayer(this.imgPlayer2.height / 2, this.imgPlayer2)
      this.binderHp1 = this.player1.hp
      this.binderHp2 = this.player2.hp
    },
    attachListener: function () {
      document.getElementById('button_home').onclick = this.onHome
      document.getElementById('button_help').onclick = this.onHelp
      document.getElementById('button_restart').onclick = this.reload
      document.getElementById('button_save').onclick = this.saveAllert
      if (typeof window.addEventListener !== 'undefined') {
        window.addEventListener('resize', this.reload)
        window.addEventListener('keydown', this.onKeydown)
        window.addEventListener('keyup', this.onKeyup)
      } else {
        alert('The version of your browser is too low.')
      }
    },
    dettachListener: function () {
      document.getElementById('button_home').onclick = null
      document.getElementById('button_help').onclick = null
      document.getElementById('button_restart').onclick = null
      document.getElementById('button_save').onclick = null
      if (typeof window.addEventListener !== 'undefined') {
        window.removeEventListener('resize', this.reload)
        window.removeEventListener('keydown', this.onKeydown)
        window.removeEventListener('keyup', this.onKeyup)
      } else {
        alert('The version of your browser is too low.')
      }
    },
    onKeydown: function (e) {
      if (this.gameState !== mGame.GameState.RUN && this.gameState !== mGame.GameState.PAUSE) return
      let code = e.keyCode
      switch (code) {
        case 32:// space
          this.changeState()
          break
        case 70:// f
          this.isShotting1 = true
          break
        case 87:// w
          this.player1.setUp(true)
          break
        case 65:// a
          this.player1.setLeft(true)
          break
        case 83:// s
          this.player1.setDown(true)
          break
        case 68:// d
          this.player1.setRight(true)
          break
        case 76:// l
          this.isShotting2 = true
          break
        case 38:// up
          this.player2.setUp(true)
          break
        case 37:// left
          this.player2.setLeft(true)
          break
        case 40:// down
          this.player2.setDown(true)
          break
        case 39:// right
          this.player2.setRight(true)
          break
        default:
          break
      }
    },
    onKeyup: function (e) {
      let code = e.keyCode
      switch (code) {
        case 70:// f
          this.isShotting1 = false
          break
        case 87:// w
          this.player1.setUp(false)
          break
        case 65:// a
          this.player1.setLeft(false)
          break
        case 83:// s
          this.player1.setDown(false)
          break
        case 68:// d
          this.player1.setRight(false)
          break
        case 76:// l
          this.isShotting2 = false
          break
        case 38:// up
          this.player2.setUp(false)
          break
        case 37:// left
          this.player2.setLeft(false)
          break
        case 40:// down
          this.player2.setDown(false)
          break
        case 39:// right
          this.player2.setRight(false)
          break
        default:
          break
      }
    },
    /** Resource Loader */
    createPlayer: function (y, imgPlayer) {
      return new mPlayer.Player(
        this.canvas,
        this.screenWidth / 2,
        y,
        imgPlayer,
        this.imgExplosion,
        null,
        null
      )
    },
    createPlayerBullet: function (x, y, dy, color) {
      return new mBullet.Bullet(
        this.canvas,
        x,
        y,
        0,
        dy,
        color
      )
    },
    /** Draw */
    loop: function () {
      if (this.gameState !== mGame.GameState.RUN) return
      this.ctx.clearRect(0, 0, this.screenWidth, this.screenHeight)
      this.loopPlayerSrc()
      this.loopPlayerAttack()
      this.animationIdLoop = requestAnimationFrame(this.loop)
    },
    loopPlayerSrc: function () {
      /** Player 1 */
      if (this.player1 !== null) {
        this.player1.updateCoord()
        this.player1.draw()
      }
      if (this.binderHp1 > 0 &&
        this.isShotting1 &&
        this.bullets1.length < mBullet.BulletConsts.MAX_COUNT) {
        var bullet1 = this.createPlayerBullet(
          this.player1.x,
          this.player1.y - this.player1.getImg().height / 2,
          -1,
          mBullet.BulletConsts.COLOR1
        )
        this.bullets1.push(bullet1)
      }

      /** Player 2 */
      if (this.player2 !== null) {
        this.player2.updateCoord()
        this.player2.draw()
      }
      if (this.binderHp2 > 0 &&
        this.isShotting2 &&
        this.bullets2.length < mBullet.BulletConsts.MAX_COUNT) {
        var bullet2 = this.createPlayerBullet(
          this.player2.x,
          this.player2.y + this.player2.getImg().height / 2,
          1,
          mBullet.BulletConsts.COLOR2
        )
        this.bullets2.push(bullet2)
      }
    },
    loopPlayerAttack: function () {
      /** Bullet 1 */
      for (var i = this.bullets1.length - 1; i >= 0; i--) {
        if (this.bullets1[i].show) {
          this.bullets1[i].updateCoord()
          this.bullets1[i].draw()
          if (this.checkCollision(this.player2, this.bullets1[i], mBullet.BulletConsts.SIZE)) {
            this.bullets1[i].show = false
            this.binderHp2 = this.updateHp(this.player2, 0 - mBullet.BulletConsts.ATTACK, 1)
          }
        } else {
          this.bullets1[i].release()
          this.bullets1.splice(i, 1)
        }
      }
      /** Bullet 2 */
      for (var j = this.bullets2.length - 1; j >= 0; j--) {
        if (this.bullets2[j].show) {
          this.bullets2[j].updateCoord()
          this.bullets2[j].draw()
          if (this.checkCollision(this.player1, this.bullets2[j], mBullet.BulletConsts.SIZE)) {
            this.bullets2[j].show = false
            this.binderHp1 = this.updateHp(this.player1, 0 - mBullet.BulletConsts.ATTACK, 2)
          }
        } else {
          this.bullets2[j].release()
          this.bullets2.splice(j, 1)
        }
      }
    },
    /** Task Scheduler */
    clearGameTasks: function () {
      cancelAnimationFrame(this.animationIdLoop)
      clearInterval(this.timeoutIdStop)
    },
    /** Game Helper */
    checkCollision: function (target, bullet, size) {
      let distance = (target.getImg().height + size) / 2
      let deltaX = target.x - bullet.x
      let deltaY = target.y - bullet.y
      return Math.pow(deltaX, 2) + Math.pow(deltaY, 2) <= Math.pow(distance, 2)
    },
    updateHp: function (player, hp, winner) {
      if (!player.alive) return
      player.updateHp(hp)
      if (!player.alive) {
        this.playerWin = winner
        this.onStop()
      }
      return player.hp
    },
    /** Game Event */
    changeState: function () {
      if (this.gameState === mGame.GameState.PAUSE) {
        this.onResume()
      } else if (this.gameState === mGame.GameState.RUN) {
        this.onPause()
      }
    },
    clear: function () {
      this.gameState = mGame.GameState.STOP
      /** Handler */
      this.clearGameTasks()
      /** Flag */
      this.isShotting1 = false
      this.isShotting2 = false
      /** Src */
      if (this.player1 !== null) {
        this.player1.release()
        this.player1 = null
      }
      if (this.player2 !== null) {
        this.player2.release()
        this.player2 = null
      }
      for (var i = 0; i < this.bullets1.length; i++) {
        this.bullets1[i].release()
      }
      this.bullets1 = []
      for (var j = 0; j < this.bullets2.length; j++) {
        this.bullets2[j].release()
      }
      this.bullets2 = []
      /** Env */
      this.canvas = null
      this.ctx = null
    },
    reload: function () {
      this.clear()
      this.onPrepare()
      this.onStart()
    },
    /** Menu */
    onHome: function () {
      this.dettachListener()
      this.clear()
      this.$router.push('/')
    },
    onHelp: function () {
      this.onPause()
      this.modalHelp = true
    },
    onResult: function (result) {
      this.modalResult = false
      if (result) {
        this.reload()
      } else {
        this.$router.push({
          path: `/`
        })
      }
    },
    saveAllert: function () {
      this.onPause()
      this.modalSave = true
    },
    onSave: function (save) {
      if (!save) {
        this.onResume()
        this.modalSave = false
        return
      }
      var saveData = this.$data
      delete saveData.canvas
      delete saveData.ctx
      delete saveData.imgPlayer1
      delete saveData.imgPlayer2
      delete saveData.imgExplosion
      delete saveData.colController
      delete saveData.dataController
      delete saveData.player1.ctx
      delete saveData.player1.imgAlive
      delete saveData.player1.imgExplosion
      delete saveData.player2.ctx
      delete saveData.player2.imgAlive
      delete saveData.player2.imgExplosion
      saveData.bullets1.forEach(element => {
        delete element.ctx
      })
      saveData.bullets2.forEach(element => {
        delete element.ctx
      })
      // console.log(JSON.stringify(saveData))
      localStorage.setItem('battle', JSON.stringify(saveData))
      this.$router.push({
        path: `/`
      })
    }
  }
}
</script>

<style scoped>
#canvas_game {
  width: 100%;
  z-index: -1;
}

.div_progress {
  width: auto;
  height: auto;
  position: absolute;
  left: 0;
  top: 50%;
  transform: translate(0, -50%);
}

.progress_hp {
  width: 200px;
  margin: 5px 0;
  display: block;
  font-size: 16px;
  color: #ff0000;
  opacity: 0.8;
}

.h1_modal {
  color: #2db7f5;
  text-align: center;
  font-size: 30px;
}

.p_modal {
  margin: 8px;
  font-size: 18px;
  color: #515a6e;
}
</style>
