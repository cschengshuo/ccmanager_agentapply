<template>
  <div>
    <form novalidate class="md-layout" @submit.prevent="validateUser">
      <md-card class="md-layout-item md-size-50 md-small-size-100">
        <md-card-header>
          <md-card-header-text>
            <div class="md-title">代理申请</div>
            <div class="md-subhead">98卡管家</div>
          </md-card-header-text>

          <md-card-media>
            <img src="./assets/logo.png" alt="98卡管家">
          </md-card-media>
        </md-card-header>

        <md-card-content>
          <md-field :class="getValidationClass('name')">
            <label for="name">姓名</label>
            <md-input name="name" id="name" v-model="form.name" :disabled="sending" />
            <span class="md-error" v-if="!$v.form.name.required">请输入姓名</span>
            <span class="md-error" v-else-if="!$v.form.name.maxlength">姓名超过最大长度</span>
          </md-field>

          <md-field :class="getValidationClass('company')">
            <label for="company">公司（选填）</label>
            <md-input id="company" name="company" v-model="form.company" :disabled="sending" />
            <span class="md-error" v-if="!$v.form.company.maxlength">公司名超过最大长度</span>
          </md-field>

          <md-field :class="getValidationClass('phone')">
            <label for="phone">手机</label>
            <md-input name="phone" id="phone" v-model="form.phone" :disabled="sending" />
            <span class="md-error" v-if="!$v.form.phone.required">请输入手机号</span>
            <span class="md-error" v-else-if="!$v.form.phone.isPhoneNumber">请输入正确的手机号</span>
          </md-field>

          <md-field :class="getValidationClass('captcha')">
            <label for="captcha">验证码</label>
            <md-input name="captcha" id="captcha" v-model="form.captcha" :disabled="sending" />
            <md-button class="md-primary" @click="sendCaptcha" :disabled="canSendCaptcha">{{ text }}</md-button>
            <span class="md-error" v-if="!$v.form.captcha.required">请输入手机验证码</span>
            <span class="md-error" v-else-if="!$v.form.captcha.numeric">手机验证码内容应为数字</span>
            <span class="md-error" v-else-if="!$v.form.captcha.minlength">手机验证码应为6位数字</span>
            <span class="md-error" v-else-if="!$v.form.captcha.maxlength">手机验证码应为6位数字</span>
          </md-field>
        </md-card-content>

        <md-progress-bar md-mode="indeterminate" v-if="sending" />

        <md-card-actions>
          <md-button type="submit" class="md-primary md-raised" :disabled="sending">点击提交</md-button>
        </md-card-actions>
      </md-card>

      <md-snackbar :md-active.sync="showTooltip">{{ tooltip }}</md-snackbar>
    </form>
  </div>
</template>

<script>
import {
  required,
  minLength,
  maxLength,
  numeric
} from 'vuelidate/lib/validators'

const reg = /^(13[0-9]|14[579]|15[0-3,5-9]|16[6]|17[0135678]|18[0-9]|19[89])\d{8}$/

export default {
  name: 'FormValidation',
  data: () => ({
    form: {
      name: null,
      company: null,
      phone: null,
      captcha: null
    },
    sending: false,
    showTooltip: false,
    tooltip: '',
    time: 0,
    sendedCaptcha: false
  }),
  validations: {
    form: {
      name: {
        required,
        maxLength: maxLength(15)
      },
      company: {
        maxLength: maxLength(35)
      },
      phone: {
        required,
        isPhoneNumber (value) {
          return reg.test(value)
        }
      },
      captcha: {
        required,
        numeric,
        minLength: minLength(6),
        maxLength: maxLength(6)
      }
    }
  },
  methods: {
    getValidationClass (fieldName) {
      const field = this.$v.form[fieldName]

      if (field) {
        return {
          'md-invalid': field.$invalid && field.$dirty
        }
      }
    },
    clearForm () {
      this.$v.$reset()
      this.form.name = null
      this.form.company = null
      this.form.phone = null
      this.form.captcha = null
    },
    sendCaptcha () {
      const me = this
      if (reg.test(this.form.phone)) {
        this.time = 60
        this.timer()
        this.$http.post('CardSteward98/user/sendSmsCode', { phone: this.form.phone, kindCode: '5' })
          .then(function (response) {
            me.tooltip = '验证码发送成功'
            me.showTooltip = true
          }).catch(function (error) {
            console.log(error)
            me.time = 0
            me.tooltip = '验证码发送失败'
            me.showTooltip = true
          })
      } else {
        this.tooltip = '请先输入正确的手机号'
        this.showTooltip = true
      }
    },
    timer () {
      if (this.time > 0) {
        this.time--
        setTimeout(this.timer, 1000)
      }
    },
    saveUser () {
      const me = this
      this.sending = true

      this.$http.post('CardSteward98/user/agentApply', {
        phone: this.form.phone,
        smsCode: this.form.captcha,
        name: this.form.name,
        company: this.form.company
      }).then(function (response) {
        if (response) {
          if (response.data.resultStatus === '200') {
            me.tooltip = '用户' + me.form.name + '的代理申请提交成功'
            me.showTooltip = true
            me.clearForm()
          } else {
            me.tooltip = response.data.resultMessage
            me.showTooltip = true
          }
        }
      }).catch(function (error) {
        console.log(error)
        me.tooltip = '代理申请提交失败'
        me.showTooltip = true
      }).then(function () {
        me.sending = false
      })
    },
    validateUser () {
      this.$v.$touch()

      if (!this.$v.$invalid) {
        this.saveUser()
      }
    }
  },
  computed: {
    text: function () {
      return this.time > 0 ? this.time + 's 后重新获取' : '获取验证码'
    },
    canSendCaptcha: function () {
      return this.time > 0
    }
  }
}
</script>

<style lang="scss" scoped>
.md-progress-bar {
  position: absolute;
  top: 0;
  right: 0;
  left: 0;
}
</style>

<style>
body {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}
</style>
