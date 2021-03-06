<template>
    <div @mouseenter="enter" @mouseleave="leave" @touchstart="touchStart" @touchmove="touchMove" @touchend="touchEnd">
        <div ref="wrapper" :style="pos">
            <slot></slot>
        </div>
    </div>
</template>
<script>
  require('comutils/animationFrame')
  const arrayEqual = require('comutils/arrayEqual')
  export default {
    data () {
      return {
        yPos: 0,
        delay: 0,
        reqFrame: null
      }
    },
    props: {
      data: {
        type: Array,
        default: []
      },
      classOption: {
        type: Object,
        default: {}
      }
    },
    computed: {
      pos () {
        return {transform: `translate(0,${this.yPos}px)`, transition: `all ease-in ${this.delay}ms`}
      },
      defaultOption () {
        return {
          step: 1, //步长
          limitMoveNum: 5, //启动无缝滚动最小数据数
          hoverStop: true, //是否启用鼠标hover控制
          direction: 1, //1 往上 0 往下
          openWatch: true, //开启data实时监听
          singleHeight: 0, //单条数据高度有值hoverStop关闭
          waitTime: 1000 //单步停止等待时间
        }
      },
      options () {
        return Object.assign({}, this.defaultOption, this.classOption)
      },
      moveSwitch () {
        return this.data.length < this.options.limitMoveNum
      }
    },
    methods: {
      touchStart (e) {
        if (!this.options.openWatch) return
        let timer
        let touch = e.targetTouches[0] //touches数组对象获得屏幕上所有的touch，取第一个touch
        this.startPos = {
          x: touch.pageX,
          y: touch.pageY
        } //取第一个touch的坐标值
        this.startPosY = this.yPos //记录touchStart时候的posY
        if (!!this.options.singleHeight) {
          if (timer) clearTimeout(timer)
          timer = setTimeout(() => {
            cancelAnimationFrame(this.reqFrame)
          }, this.options.waitTime + 20)
        } else {
          cancelAnimationFrame(this.reqFrame)
        }
      },
      touchMove (e) {
        //当屏幕有多个touch或者页面被缩放过，就不执行move操作
        if (!this.options.openWatch || e.targetTouches.length > 1 || e.scale && e.scale !== 1) return
        let touch = e.targetTouches[0]
        this.endPos = {
          x: touch.pageX - this.startPos.x,
          y: touch.pageY - this.startPos.y
        }
        let direction = Math.abs(this.endPos.x) < Math.abs(this.endPos.y) ? 1 : 0 //direction，1表示纵向滑动，0为横向滑动
        if (direction === 1) {
          event.preventDefault(); //阻止触摸事件的默认行为，即阻止滚屏
          this.yPos = this.startPosY + this.endPos.y
        }
      },
      touchEnd () {
        if (!this.options.openWatch) return
        let timer
        let direction = this.options.direction
        this.delay = 50
        if (direction === 1) {
          if (this.yPos > 0) this.yPos = 0
        } else {
          let h = this.$refs.wrapper.offsetHeight / 2 * -1
          if (this.yPos < h) this.yPos = h
        }
        if (timer) clearTimeout(timer)
        timer = setTimeout(() => {
          this.delay = 0
          this._move()
        }, this.delay)
      },
      enter () {
        if (!this.options.openWatch || !!this.options.singleHeight || !this.options.hoverStop || this.moveSwitch) return
        cancelAnimationFrame(this.reqFrame)
      },
      leave () {
        if (!this.options.openWatch || !!this.options.singleHeight || !this.options.hoverStop || this.moveSwitch) return
        this._move()
      },
      _move () {
        this.reqFrame = requestAnimationFrame(
          () => {
            let timer
            let h = this.$refs.wrapper.offsetHeight / 2
            let direction = this.options.direction
            if (direction === 1) {
              if (Math.abs(this.yPos) >= h) this.yPos = 0
            } else {
              if (this.yPos >= 0) this.yPos = h * -1
            }
            if (direction === 1) {
              this.yPos -= this.options.step
            } else {
              this.yPos += this.options.step
            }
            if (!!this.options.singleHeight) {
              if (Math.abs(this.yPos) % this.options.singleHeight === 0) {
                if (timer) clearTimeout(timer)
                timer = setTimeout(() => {
                  this._move()
                }, this.options.waitTime)
              } else {
                this._move()
              }
            } else {
              this._move()
            }
          }
        )
      },
      _initMove () {
        if (this.moveSwitch) {
          cancelAnimationFrame(this.reqFrame)
          this.yPos = 0
        } else {
          this.$emit('copy-data')
          if (this.options.direction !== 1) {
            setTimeout(() => {
              this.yPos = this.$refs.wrapper.offsetHeight / 2 * -1
            }, 20)
          }
          this._move()
        }
      }
    },
    mounted () {
      this._initMove()
    },
    watch: {
      data (newData, oldData) {
        if (!this.options.openWatch) return
        if (!arrayEqual(newData, oldData.concat(oldData))) {
          cancelAnimationFrame(this.reqFrame)
          this._initMove()
        }
      }
    }
  }
</script>
