<template>
  <b-modal :id="modalID"
           ref="commModal"
           centered
           title="配置串口信息"
           size="lg"
           @ok.prevent="onCommOk"
           @shown="onShown">
    <form>
      <b-form-group label="连接方式">
        <b-form-radio-group v-model="connectionType" name="connectionType">
          <b-form-radio value="local">本地连接</b-form-radio>
          <b-form-radio value="remote">远程连接</b-form-radio>
        </b-form-radio-group>
      </b-form-group>
      <b-card no-body v-show="connectionType === 'remote'">
        <b-container fluid>
          <b-row>
            <b-col sm="3" class="mt-2"><label>远端串口地址</label></b-col>
            <b-col sm="9">
              <b-form-input
                class="mb-2 mt-2"
                size="sm"
                v-model="remoteIP"
                type="text"
                :state="IPState"
                aria-describedby="IPFeedback"
                placeholder="例: 192.168.1.1">
              </b-form-input>
              <b-form-invalid-feedback id="IPFeedback">
                IP地址不正确
              </b-form-invalid-feedback>
            </b-col>
            <b-col sm="3"><label>远端串口端口</label></b-col>
            <b-col sm="9">
              <b-form-input
                class="mb-2"
                size="sm"
                v-model="remotePort"
                type="text"
                :state="portState"
                aria-describedby="PortFeedback"
                placeholder="例: 8848">
              </b-form-input>
              <b-form-invalid-feedback id="PortFeedback">
                IP端口不正确
              </b-form-invalid-feedback>
            </b-col>
            <b-col sm="3"><label>远端串口名称</label></b-col>
            <b-col sm="9">
              <b-form-input
                class="mb-2"
                size="sm"
                v-model="remoteComm"
                type="text"
                :formatter="commFormat"
                placeholder="例: COM1">
              </b-form-input>
            </b-col>
          </b-row>
        </b-container>
      </b-card>
      <b-card v-show="connectionType === 'local'">
        <b-form inline>
          <label class="mr-sm-2">串口</label>
          <b-form-select class="mb-2 mr-sm-2 mb-sm-0"
                         v-model="commSelected"
                         :options="commList"
                         size='sm'>
          </b-form-select>
          <label class="mr-sm-2">波特率</label>
          <b-form-select class="mb-2 mr-sm-2 mb-sm-0"
                         v-model="baudrateSelected"
                         :options="baudrates"
                         size='sm'>
          </b-form-select>
          <label class="mr-sm-2">数据位</label>
          <b-form-select class="mb-2 mr-sm-2 mb-sm-0"
                         v-model="databitSelected"
                         :options="databits"
                         size='sm'>
          </b-form-select>
          <label class="mr-sm-2">停止位</label>
          <b-form-select class="mb-2 mr-sm-2 mb-sm-0"
                         v-model="stopbitSelected"
                         :options="stopbits"
                         size='sm'>
          </b-form-select>
          <label class="mr-sm-2">校验位</label>
          <b-form-select class="mb-2 mr-sm-2 mb-sm-0"
                         v-model="paritySelected"
                         :options="parity"
                         size='sm'>
          </b-form-select>
        </b-form>
      </b-card>
      <br/>
      <b-form-checkbox v-model="advancedComm" class="mb-2 mr-sm-2 mb-sm-0">显示高级选项</b-form-checkbox>
      <b-card no-body v-show="advancedComm === true">
        <b-form inline>
          <b-form-checkbox class="ml-2 mb-2 ml-sm-2 mr-sm-0 mb-sm-0"
                         v-model="timestampEnabled"
                         v-b-tooltip.hover title="在每行消息开头添加时间戳"
                         value="true"
                         size='sm'>
            时间戳
          </b-form-checkbox>
          <b-form-checkbox class="mb-2 ml-sm-2 mr-sm-0 mb-sm-0"
                         v-model="highlightEnabled"
                         v-b-tooltip.hover title="消息高亮，规则可定制，具体见设置"
                         value="true"
                         size='sm'>
            消息高亮
          </b-form-checkbox>
          <b-form-checkbox class="mb-2 ml-sm-2 mr-sm-0 mb-sm-0"
                         v-model="implicitCarriageEnabled"
                         v-b-tooltip.hover title="当单独出现\n时，自动在前面补足\r，补足后\n变成\r\n，成对的\r\n不受影响"
                         value="true"
                         size='sm'>
            自动补足"\r"
          </b-form-checkbox>
          <b-form-checkbox class="mb-2 ml-sm-2 mr-sm-0 mb-sm-0"
                         v-model="implicitLineFeedEnabled"
                         v-b-tooltip.hover title="当单独出现\r时，自动在后面补足\n，补足后\r变成\r\n，成对的\r\n不受影响"
                         value="true"
                         size='sm'>
            自动补足"\n"
          </b-form-checkbox>
          <b-form-checkbox class="mb-2 ml-sm-2 mr-sm-0 mb-sm-0"
                         v-model="localEchoEnabled"
                         v-b-tooltip.hover title="一般设备端会回显消息，如果不支持可以开启该功能"
                         value="true"
                         size='sm'>
            本地消息回显
          </b-form-checkbox>
          <b-form-checkbox class="mb-2 ml-sm-2 mr-sm-0 mb-sm-0"
                         v-model="localEchoEnabled"
                         v-b-tooltip.hover title="一般设备端会提供命令记忆功能，如果不支持可以开启该功能，使用up和down方向键选择"
                         value="true"
                         size='sm'>
            本地命令记录
          </b-form-checkbox>
        </b-form>
        <b-form inline>
          <b-form-checkbox
                   class="ml-2 mb-2 ml-sm-2 mr-sm-0 mb-sm-0"
                   v-b-tooltip.hover title="以websocket方式在局域网中共享该终端，地址就是本机IP地址+端口号+共享名称，如:192.168.1.100:8848/COM3"
                   size="sm"
                   aria-label="Checkbox for following text input"
                   v-model="sharedOverWebsocket">
            远程共享该串口
          </b-form-checkbox>
          <b-form-input size="sm" type="text" placeholder="共享名称(默认当前串口号)" v-show="sharedOverWebsocket === true"/>
        </b-form>
      </b-card>
    </form>
  </b-modal>
</template>

<script>
  import SerialPort from 'serialport'
  import {mapState} from 'vuex'
  
  export default {
    name: 'commConfig',
    props: [
      'modalID'
    ],
    data () {
      return {
        commSelected: null,
        advancedComm: false,
        baudrates: [
          4800,
          9600,
          14400,
          19200,
          38400,
          57600,
          115200
        ],
        databits: [
          8,
          7,
          6,
          5
        ],
        stopbits: [
          1,
          2
        ],
        parity: [
          'none',
          'even',
          'mark',
          'odd',
          'space'
        ],
        connectionType: 'local',
        remoteIP: null,
        remotePort: null,
        remoteComm: null,
        baudrateSelected: 115200,
        databitSelected: 8,
        stopbitSelected: 1,
        paritySelected: 'none',
        timestampEnabled: true,
        highlightEnabled: true,
        implicitCarriageEnabled: true,
        implicitLineFeedEnabled: true,
        localEchoEnabled: true,
        localHistoryEnabled: true,
        sharedOverWebsocket: false
      }
    },
    // mounted () {
    //   console.log(this.$store.state.terminal)
    // },
    computed: {
      ...mapState({
        terminals: state => state.terminal.terminals,
        commList: state => state.terminal.commList,
        tabs: state => state.terminal.tabs
      }),
      IPState () {
        if (!this.remoteIP) return true
        let ip = this.remoteIP.split('.')
        if (!ip || ip.length !== 4 || ip[3] === '' ||
            !/^\d+$/.test(ip[0] + ip[1] + ip[2] + ip[3])) {
          return false
        }
        return true
      },
      portState () {
        if (!this.remotePort) return true
        return /^\d+$/.test(this.remotePort)
      }
    },
    methods: {
      commFormat (value, event) {
        return value.toUpperCase()
      },
      getComm () {
        return new Promise((resolve, reject) =>
          SerialPort.list()
            .then(ports => {
              // ports = [{'comName': 'COM1'}, {'comName': 'COM2'}, {'comName': 'COM3'}]
              ports.forEach(port => {
                if (this.commList.indexOf(port.comName) < 0) {
                  this.$store.commit('ADD_COMM', port.comName)
                }
              })
              resolve()
            })
            .catch(reject)
        )
      },
      checkComm (comm) {
        return new Promise((resolve, reject) => {
          const port = new SerialPort(comm, {
            autoOpen: false // to catch opening error
          })
          port.open(err => {
            if (err === null) {
              port.close(err => { if (err === null) resolve() })
            } else {
              reject(err)
            }
          })
        })
      },
      onShown (evt) {
        this.getComm()
          .catch(alert)
      },
      onCommOk (evt) {
        if (!this.commSelected || !this.baudrateSelected ||
            !this.databitSelected || !this.stopbitSelected ||
            !this.paritySelected) {
          alert('Please choose valid config for serial port')
          return
        }
        if (this.tabs.indexOf(this.commSelected) < 0) {
          let idx = this.commList.indexOf(this.commSelected)
          if (idx < 0) {
            console.log(this.commList, this.commSelected)
            return
          }

          this.checkComm(this.commSelected).then(() => {
            this.$store.commit('ADD_TERMINAL', {
            //   pid: idx,
              comm: this.commSelected,
              baudRate: this.baudrateSelected,
              dataBits: this.databitSelected,
              stopBits: this.stopbitSelected,
              parity: this.paritySelected,
              timestampEnabled: this.timestampEnabled,
              highlightEnabled: this.highlightEnabled,
              implicitCarriageEnabled: this.implicitCarriageEnabled,
              implicitLineFeedEnabled: this.implicitLineFeedEnabled,
              localEchoEnabled: this.localEchoEnabled,
              localHistoryEnabled: this.localHistoryEnabled,
              history: [''],
              historyIdx: 0,
              historyRecallIdx: null
            })
            this.$store.commit('ADD_TAB', idx)
            this.$refs.commModal.hide()
          }, console.log)
        } else {
          alert(`It's already opened, choose another one`)
        }
        // this.$emit('ok', evt)
      }
    }
  }
</script>