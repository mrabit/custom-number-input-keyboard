<template>
  <div class="input-container" @click.stop="handleClickOpen()">
    <div class="input" @click.stop="handleClickOpen()" ref="input">
      <span class="placeholder" v-show="!value" ref="placeholder">{{placeholder}}</span>
      <span class="cursor" v-show="postion === -1"></span>
      <template v-for="(item,index) in value">
        <span :class="{cursor:index===postion}" @click.stop="handleClickOpen(index)">{{item}}</span>
      </template>
    </div>
  </div>
</template>
<script>
import keyboard from "./keyboard";
import Vue from "vue";

export default {
  name: "custom-input",
  data() {
    return {
      postion: null,
      show: false
    };
  },
  props: {
    // 自定义input值,需要分割每个字符到span标签内,所以type=String
    value: {
      type: String,
      default: ""
    },
    placeholder: {
      type: String,
      default: "placeholder"
    },
    // 是否判断自定义input位置被自定义键盘覆盖
    transform: {
      type: Boolean,
      default: true
    },
    // 自定义input最大值设置
    max: {
      type: [String, Number],
      default: 9999999999.99
    },
    // 小数点后保留位数
    toFixed: {
      type: Number,
      default: 2
    }
  },
  computed: {
    maxNumber() {
      return this.max.toString();
    }
  },
  methods: {
    // 自定义键盘展开事件
    handleClickOpen: function(index = this.value.length) {
      this.$root.customInput.input.postion = null;
      this.$root.customInput.input = this;
      if (this.transform) {
        // 当前窗口高度
        let winHeight = window.innerHeight;
        // 当前自定义input高度
        let inputHeight = this.$el.offsetHeight;
        // 当前自定义input距离顶部高度
        let inputTop = this.$el.offsetTop;
        // 当前自定义键盘高度
        let keyboardHeight = this.$root.customInput.keyboard.$el.offsetHeight;
        // 当前自定义input距离底部高度
        let inputBottom = winHeight - inputHeight - inputTop;
        inputBottom = inputBottom < 0 ? inputBottom * -1 : inputBottom;
        // 当前自定义input距离底部高度小于当前自定义键盘高度,需要滚动
        if (inputBottom < keyboardHeight) {
          this.$parent.$el.style.transition = "all 0.3s";
          this.$parent.$el.style.transform = `translate3d(0px, -${keyboardHeight}px, 0px)`;
        }
      }
      this.$emit("onKeyboardOpen", this, this.$root.customInput.keyboard);
      this.show = true;
      this.postion = index - 1;
      // object是引用传递,原数据更新,自定义键盘内的数据就会更新
      Object.assign(this.$root.customInput.keyboard, {
        value: this.$root.customInput.input.$data
      });
    }
  },
  watch: {
    value: {
      immediate: true,
      handler(val, oldVal) {
        if (val != oldVal) {
          if (parseFloat(val) > parseFloat(this.max)) {
            console.warn("maximum is: " + this.max, this);
            this.$emit("input", "" + this.max);
          }
        }
      }
    }
  },
  // 自定义input释放事件
  beforeDestroy() {
    // 当前页面不存在自定义input,释放键盘
    if (
      this.$root.customInput &&
      document.querySelectorAll(".input-container").length <= 0
    ) {
      this.$root.customInput.keyboard.$destroy();
      document.body.removeChild(this.$root.customInput.keyboard.$el);
      delete this.$root.customInput.keyboard;
    }
  },
  created() {
    // 当前已存在自定义键盘
    if (this.$root.customInput && this.$root.customInput.keyboard) return false;
    // 实例化自定义键盘
    let vmKeyboard = new (Vue.extend(keyboard))({
      el: document.createElement("div")
    });
    this.$root.customInput = {
      keyboard: vmKeyboard,
      input: this
    };
    document.body.appendChild(vmKeyboard.$el); // 添加到dom
    // 自定义键盘点击事件
    vmKeyboard.$on("handleClick", val => {
      let input = this.$root.customInput.input;

      let isDot = val === ".";
      // 已存在小数点
      if (input.value.indexOf(".") >= 0 && isDot) {
        return false;
      }

      let temp = input.value.split("");
      // 当前光标位置+1位插入值
      temp.splice(input.postion + 1, 0, val);
      let newValue = temp.join("");

      // 判断新值大于预定值,不执行赋值操作
      if (parseFloat(newValue) > parseFloat(input.maxNumber)) {
        return false;
      }

      // 最大值判断优于小数点位数判断
      // 不满足当前正则(禁止直接输入小数点,小数点后保留位数),不执行赋值操作
      let toFixed = input.toFixed;
      let reg = new RegExp(`^(([1-9]{1}\\d*)|(0{1}))(\\.\\d{0,${toFixed}})?$`);
      if (!reg.test(newValue)) {
        return false;
      }

      // 赋值,当前光标位置+1
      input.$emit("input", newValue);
      input.postion++;
    });
    // 自定义键盘点击删除事件
    vmKeyboard.$on("handleClickDelete", _ => {
      let input = this.$root.customInput.input;
      // 当前光标位置为-1(首位), 不进行任何删除操作
      if (input.postion < 0) return false;
      let temp = input.value.split("");
      temp.splice(input.postion, 1);
      // 赋值,当前光标位置-1
      input.$emit("input", temp.join(""));
      input.postion--;
    });
    // 自定义键盘点击完成事件
    vmKeyboard.$on("close", _ => {
      let input = this.$root.customInput.input;
      input.$emit("close", input.value);
      input.$parent.$el.style.transform = "";
      input.show = false;
      input.postion = null;
    });
    // object是引用传递,原数据更新,自定义键盘内的数据就会更新
    Object.assign(vmKeyboard, {
      value: this.$root.customInput.input.$data
    });
  }
};
</script>
<style lang="less" scoped>
.input-container {
  // width: 100%;
  padding: 10px 15px;
  display: flex;
  align-items: center;
  overflow: hidden;
  .input {
    width: 100%;
    padding-left: 8px;
    line-height: 35px;
    height: 35px;
    font-size: 30px;
    font-family: DINAlternate-Bold;
    font-weight: bold;
    flex: auto;
    text-align: left;
    display: flex;
    justify-content: flex-start;
    align-items: center;
    position: relative;
    .placeholder {
      position: absolute;
      left: 8px;
      color: #bccad8;
      font-size: 18px;
      font-weight: initial;
    }
    span {
      display: block;
    }
    .cursor {
      position: relative;
      height: 35px;
      &::after {
        content: "";
        animation: blink 1s infinite steps(1, start);
        position: absolute;
        width: 2px;
        height: 30px;
        right: 0;
        top: 2.5px;
        background: rgb(0, 53, 237);
        border-radius: 2px;
      }
    }
  }
}
@keyframes blink {
  0%,
  100% {
    background: transparent;
  }
  50% {
    background: rgb(0, 53, 237);
  }
}
</style>
