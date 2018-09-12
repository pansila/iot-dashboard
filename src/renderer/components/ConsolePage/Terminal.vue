<template>
  <div class="">
    <resize-sensor @resize="onResize"></resize-sensor>
  </div>
</template>

<script>
import {mapState} from 'vuex'
import {Terminal} from 'xterm'
import * as fit from 'xterm/lib/addons/fit/fit'
// import * as attach from 'xterm/lib/addons/attach/attach'
import 'xterm/dist/xterm.css'
import resizesensor from '@components/ResizeSensor'
import SerialPort from 'serialport'
import Highlighter from '@utils/Highlighter'
import {TimestampPrefix, LineParser} from '@utils/Common.js'
// import {TimestampPrefix, LineBreaker, LineParser} from '@utils/Common.js'
import {PassThrough} from 'stream'

Terminal.applyAddon(fit)
// Terminal.applyAddon(attach)

export default {
  name: 'Console',
  props: [
    'containerID',
    'pid'
  ],
  computed: {
    ...mapState({
      terminals: state => state.terminal.terminals
    }),
    terminal () {
      return this.terminals[this.pid]
    },
    history () {
      return this.terminal.history
    },
    historyIdx () {
      return this.terminal.historyIdx
    },
    commandIndex () {
      return this.terminal.historyRecallIdx
    }
  },
  watch: {
    commandIndex (newVal, oldVal) {
      if (newVal !== null) {
        this.term.write('\b'.repeat(this.input.length) +
                        ' '.repeat(this.input.length) +
                        '\b'.repeat(this.input.length))
        this.input = this.history[newVal]
        this.term.write(this.input)
      } else if (this.input !== '') {
        this.term.write('\b'.repeat(this.input.length))
        this.serialport.write(this.input + '\r')
        this.input = ''
      }
    }
  },
  data () {
    return {
      term: null,
      // terminalSocket: null
      serialport: null,
      input: '',
      lookupHistory: false,
      // promptOffset: 0,
      highlightOptions: {
        highlightOptions: [
          {
            colorText: 'red.bold',
            patternArray: ['error', 'fail', 'failed'],
            modifiers: {}
          },
          {
            colorText: 'yellow.bold',
            patternArray: ['warn', 'warning'],
            modifiers: {}
          },
          {
            colorText: 'blue.bold',
            patternArray: ['succeed', 'succeeded', 'successfully'],
            modifiers: {}
          },
          {
            colorText: 'green.bold',
            patternArray: ['\\bstart', '\\bstop', '\\bcreate', '\\bcreated', '\\bcomplete', '\\bcompleted', '\\bfinish', '\\bfinished', '\\bend'],
            modifiers: {}
          },
          {
            colorText: 'bgBlue',
            patternArray: ['\\d+(\\.\\d+){3}', '[0-9a-fA-F]{2}(:[0-9a-fA-F]{2}){3,5}', '0x[0-9a-fA-F]+'],
            modifiers: {}
          }
        ],
        caseSensitive: false,
        defaultStyle: 'white'
      }
    }
  },
  methods: {
    runRealTerminal () {
      console.log('webSocket is finished')
    },
    errorRealTerminal () {
      console.log('error')
    },
    closeRealTerminal () {
      console.log('close')
    },
    setupTerminal () {
      let terminalContainer = document.getElementById(this.containerID)
      this.term = new Terminal({
        rendererType: 'dom'
      })
      this.term.open(terminalContainer)
      // open websocket
      // this.terminalSocket = new WebSocket('ws://127.0.0.1:3000/terminals/')
      // this.terminalSocket.onopen = this.runRealTerminal
      // this.terminalSocket.onclose = this.closeRealTerminal
      // this.terminalSocket.onerror = this.errorRealTerminal
      // this.term.attach(this.terminalSocket)
      this.term.fit()
      this.term._initialized = true
      this.term.on('key', (data, ev) => {
        const printable = !ev.altKey && !ev.altGraphKey && !ev.ctrlKey && !ev.metaKey

        if (ev.code === 'ArrowUp' || ev.code === 'ArrowDown') {
          if (!this.terminal.localHistoryEnabled) {
            this.serialport.write(data)
            return
          }

          if (!this.lookupHistory && this.historyIdx === this.history.length - 1 && this.input) {
            this.$store.commit('ADD_HISTORY', {pid: this.pid, input: this.input})
          }
          if (ev.code === 'ArrowDown') {
            if (this.historyIdx !== this.history.length - 1) {
              this.$store.commit('INCREMENT_HISTORY_IDX', this.pid)
            } else if (this.input !== '') {
              this.term.write('\b'.repeat(this.input.length) +
                              ' '.repeat(this.input.length) +
                              '\b'.repeat(this.input.length))
              this.input = ''
              return
            } else {
              return
            }
          }
          this.lookupHistory = true
          this.term.write('\b'.repeat(this.input.length) +
                          ' '.repeat(this.input.length) +
                          '\b'.repeat(this.input.length))
          this.input = this.history[this.historyIdx]
          this.term.write(this.input)
          if (ev.code === 'ArrowUp' && this.historyIdx > 0) {
            this.$store.commit('DECREMENT_HISTORY_IDX', this.pid)
          }
          return
        }

        if (ev.code === 'ArrowLeft' || ev.code === 'ArrowRight') {
          return
        }

        if (ev.keyCode === 13) {
          if (this.terminal.localHistoryEnabled) {
            if (this.input) {
              this.$store.commit('ADD_HISTORY', {pid: this.pid, input: this.input})
            }
            this.lookupHistory = false
          }

          if (this.terminal.localEchoEnabled) {
            this.term.write('\b'.repeat(this.input.length))
            this.serialport.write(this.input + '\r')
          } else {
            this.serialport.write('\r')
            // this.term.write('\r')
          }
          this.input = ''
        } else if (ev.keyCode === 8) {
          if (this.input.length > 0) {
            this.term.write('\b \b')
            this.input = this.input.slice(0, -1)
          }
        } else if (printable) {
          this.input += data
          if (this.terminal.localEchoEnabled) {
            this.term.write(data)
          } else {
            this.serialport.write(data)
          }
        }
      })
    },
    setupSerialport () {
      return new Promise((resolve, reject) => {
        SerialPort.list()
          .then(ports => {
            ports.forEach(port => {
              if (this.terminal.comm === port.comName) {
                const port = new SerialPort(this.terminal.comm, {
                  autoOpen: false, // to catch opening error
                  baudRate: this.terminal.baudRate,
                  dataBits: this.terminal.dataBits,
                  stopBits: this.terminal.stopBits,
                  parity: this.terminal.parity
                })
                port.open(err => {
                  if (err === null) {
                    resolve(port)
                  } else {
                    reject(err)
                  }
                })
              }
            })
          })
          .catch(reject)
      })
    },
    onResize (size) {
      if (this.term) {
        // console.log('resize to fit:', size)
        this.term.fit()
        return
      }

      this.setupSerialport()
        .then(port => {
          this.serialport = port
          this.setupTerminal()

          // const lineParser = new SerialPort.parsers.Readline({ delimiter: '\r\n' })
          const lineParser = LineParser(this.terminal.implicitCarriageEnabled, this.terminal.implicitLineFeedEnabled)
          const highlighter = this.terminal.highlightEnabled ? Highlighter(this.highlightOptions) : new PassThrough()
          const timestampPrefix = this.terminal.timestampEnabled ? TimestampPrefix() : new PassThrough()
          // const lineBreaker = this.terminal.implicitCarriageEnabled ? LineBreaker() : new PassThrough()

          port.on('close', e => { this.serialport = null; console.log('Close', e) })
          port.on('error', alert)
          port
            .pipe(lineParser)
            .pipe(highlighter)
            .pipe(timestampPrefix)
            // .pipe(lineBreaker)
            .on('data', data => {
              // console.log(Array.from(data).map(ch => ch.charCodeAt()))
              this.term.write(data)
            })
        })
        .catch(console.log)
    }
  },
  mounted () {
    // this.setupTerminal()
  },
  beforeDestroy () {
    // this.term.detach(this.terminalSocket)
    // this.terminalSocket.close()
    if (this.serialport) this.serialport.close()
    if (this.term) this.term.destroy()
  },
  components: {
    'resize-sensor': resizesensor
  }
}
</script>

<style>
</style>