<template>
  <div>
    <a-form :form="form">
      <a-form-item label="Platform" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
        <a-radio-group name="platform" v-model="form.platform">
          <a-radio value="github">Github Pages</a-radio>
          <a-radio value="coding">Coding Pages</a-radio>
          <a-radio value="sftp">SFTP</a-radio>
        </a-radio-group>
      </a-form-item>
      <a-form-item :label="$t('domain')" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
        <a-input v-model="form.domain" placeholder="http(s)://" />
      </a-form-item>
      <template v-if="['github', 'coding'].includes(form.platform)">
        <a-form-item :label="$t('repository')" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
          <a-input v-model="form.repository" />
        </a-form-item>
        <a-form-item :label="$t('branch')" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
          <a-input v-model="form.branch" />
        </a-form-item>
        <a-form-item :label="$t('username')" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
          <a-input v-model="form.username" />
        </a-form-item>
        <a-form-item :label="$t('email')" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
          <a-input v-model="form.email" />
        </a-form-item>
        <a-form-item label="Token" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
          <a-input :type="passVisible ? '' : 'password'" v-model="form.token">
            <a-icon class="icon" slot="addonAfter" :type="passVisible ? 'eye-invisible' : 'eye'" @click="passVisible = !passVisible" />
          </a-input>
        </a-form-item>
        <a-form-item label="CNAME" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
          <a-input v-model="form.cname" />
        </a-form-item>
      </template>
      <template v-if="['sftp'].includes(form.platform)">
        <a-form-item label="Port" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
          <a-input-number v-model="form.port" />
        </a-form-item>
        <a-form-item label="Server" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
          <a-input v-model="form.server" />
        </a-form-item>
        <a-form-item label="Username" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
          <a-input v-model="form.username" />
        </a-form-item>
        <a-form-item label="Password" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
          <a-input v-model="form.password" :type="passVisible ? '' : 'password'">
            <a-icon class="icon" slot="addonAfter" :type="passVisible ? 'eye-invisible' : 'eye'" @click="passVisible = !passVisible" />
          </a-input>
        </a-form-item>
        <a-form-item label="Remote Path" :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false" help="请填写绝对路径，例如：/home/username/www/">
          <a-input v-model="form.remotePath" />
        </a-form-item>
      </template>
      <a-form-item label=" " :labelCol="formLayout.label" :wrapperCol="formLayout.wrapper" :colon="false">
        <a-button :disabled="!canSubmit" :loading="detectLoading" @click="remoteDetect" style="margin-right: 16px;">{{ $t('testConnection') }}</a-button>
        <a-button :disabled="!canSubmit" @click="submit" type="primary">{{ $t('save') }}</a-button>
      </a-form-item>
      
    </a-form>
  </div>
</template>

<script lang="ts">
import { ipcRenderer, IpcRendererEvent } from 'electron'
import { Vue, Component, Watch } from 'vue-property-decorator'
import { State } from 'vuex-class'
import ga from '../../../helpers/analytics'
import { ISetting } from '../../../interfaces/setting'

@Component
export default class BasicSetting extends Vue {
  @State('site') site!: any

  passVisible = false

  detectLoading = false

  formLayout = {
    label: { span: 6 },
    wrapper: { span: 12 },
  }

  form: ISetting = {
    platform: 'github',
    domain: '',
    repository: '',
    branch: '',
    username: '',
    email: '',
    token: '',
    cname: '',
    port: '22',
    server: '',
    password: '',
    remotePath: '',
  }

  get canSubmit() {
    const { form } = this
    const pagesPlatfomValid = ['github', 'coding'].includes(form.platform)
      && form.domain
      && form.repository
      && form.branch
      && form.username
      && form.token
    
    const sftpPlatformValid = ['sftp'].includes(form.platform)
      && form.port
      && form.server
      && form.username
      && form.password
      && form.remotePath

    return pagesPlatfomValid || sftpPlatformValid
  }

  mounted() {
    const { form, site: { setting } } = this
    console.log('setting', setting)
    Object.keys(form).forEach((key: string) => {
      form[key] = setting[key]
    })
  }

  /**
   * check form validate
   * @returns {boolean}
   */
  checkFormValid() {
    if (!['https://', 'http://'].some(d => this.form.domain.startsWith(d))) {
      this.$message.warn(this.$t('domainShouldStartsWithWarn'))
      return false
    }
    return true
  }

  submit() {
    const formValid = this.checkFormValid()
    if (!formValid) { return false }

    ipcRenderer.send('setting-save', this.form)
    ipcRenderer.once('setting-saved', (event: IpcRendererEvent, result: any) => {
      this.$bus.$emit('site-reload')
      this.$message.success(this.$t('basicSettingSuccess'))

      ga.event('Setting', 'Setting - save', { evLabel: this.form.platform })
    })
  }

  async remoteDetect() {
    ipcRenderer.send('setting-save', this.form)
    ipcRenderer.once('setting-saved', () => {
      ipcRenderer.send('app-site-reload')
      ipcRenderer.once('app-site-loaded', () => {
        this.detectLoading = true
        ipcRenderer.send('remote-detect')

        ga.event('Setting', 'Setting - detect', { evLabel: this.form.platform })
        ipcRenderer.once('remote-detected', (event: IpcRendererEvent, result: any) => {
          console.log('检测结果', result)
          this.detectLoading = false
          if (result.success) {
            this.$message.success(this.$t('connectSuccess'))

            ga.event('Setting', 'Setting - detect-success', { evLabel: this.form.platform })
          } else {
            this.$message.error(this.$t('connectFailed'))

            ga.event('Setting', 'Setting - detect-failed', { evLabel: this.form.platform })
          }
        })
      })
    })
  }

  @Watch('form.token')
  onTokenChanged(val: string) {
    this.form.token = this.form.token.trim()
  }
}
</script>


<style lang="less" scoped>
.icon {
  cursor: pointer;
}
</style>
