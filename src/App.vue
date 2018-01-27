<template>
  <div class="container-fluid" id="app">
    <img src="./assets/logo.png" class="img-fluid" id="logo" />
    <Main v-if="wallets !== null || offline" :wallets="wallets" :ripple="ripple" />

    <div class="clearfix"></div>
    <div v-if="wallets === null && !offline" class="text-center alert text-muted">
      <br /><br /><br />
      <h2>XRP Wallet</h2>
      Connecting...
    </div>
  </div>
</template>

<script>
import Main from './components/Main'
import localforage from 'localforage'
import Vue from 'vue'
import swal from 'sweetalert'

localforage.config({ name: 'kyte' })

const ripple = window.ripple
window.eventBus = new Vue({
  name: 'KyteEventBus',
  data () {
    return {
      lastLedger: {},
      lastLedgerVersion: null,
      ripple: null,
      subscribedAccounts: [],
      rippleFullHistory: null,
      connected: false,
      awaitConnected: [],
      currency: 'EUR',
      offline: false,
      fee: 0.000012,
      maxFee: 0.1,
      txQueue: []
    }
  },
  created: function () {
    this.connectFullHistoryNode()
    localforage.getItem('currency', function (err, r) {
      if (err) {}
      if (typeof r === 'string') window.eventBus.currency = r
    })
  },
  watch: {
    currency: function (a, b) {
      if (a !== b) {
        this.$emit('currencyswitch', a)
        localforage.setItem('currency', a, function (err) {
          console.log('Store currency [err?] ', err)
        })
      }
    },
    lastLedgerVersion: function (l) {
      var $app = this

      console.log('LAST LEDGER: ', l)

      var expiredTx = this.txQueue.filter(function (lQ) {
        return !lQ.txJson || (lQ.txJson.LastLedgerSequence && lQ.txJson.LastLedgerSequence < l)
      })

      var txOk = function (t, exp) {
        $app.txQueue.splice(exp, 1)
        window.getBalances()

        if (!window.lastTxSwal || window.lastTxSwal !== t.id) { // Prevent closing & opening for the same TX
          if (swal.getState().isOpen) swal.close()

          if (t.outcome.result === 'tesSUCCESS') {
            var msg = 'A ledger with your transaction just closed.'

            console.log(t)

            if (t.specification && t.specification.destination) {
              msg = 'Transaction to ' + t.specification.destination.address + ' is processed!'
            }
            if (exp.txJson) {
              msg = 'Transaction to ' + exp.txJson.Destination + ' is processed!'
            }
            swal('Transaction processed', msg, 'success')

            if (t.id) window.lastTxSwal = t.id
          } else {
            swal('Transaction (warning)', 'Transaction to ' + exp.txJson.Destination + ': ' + t.outcome.result, 'warning')
          }
        }
      }

      var txErr = function (e, exp) {
        if (swal.getState().isOpen) swal.close()
        $app.txQueue.splice(exp, 1)
        var msg = 'Failure: ' + e.message
        if (exp.txJson) {
          msg = 'Transaction to ' + exp.txJson.Destination + ': ' + e.message
        }
        swal('Transaction (error)', msg, 'error')
      }

      if (expiredTx.length > 0) {
        expiredTx.forEach(function (exp) {
          console.log('Getting TX')
          $app.ripple.getTransaction(exp.txSigned.id).then(function (t) {
            txOk(t, exp)
          }).catch(function (e) {
            if (e.name === 'MissingLedgerHistoryError' || e.message === 'noNetwork' || e.name === 'NotFoundError') {
              $app.rippleFullHistory.getTransaction(exp.txSigned.id).then(function (t) {
                txOk(t, exp)
              }).catch(function (e) {
                console.log('getTransaction @ ripple s2: ', e)
                if (e.name === 'MissingLedgerHistoryError' || e.message === 'noNetwork' || e.name === 'NotFoundError') {
                  // Retry some more
                } else {
                  txErr(e, exp)
                }
              })
            } else {
              console.log('getTransaction @ prim. server: ', e)
              txErr(e, exp)
            }
          })
        })
      }
    }
  },
  methods: {
    submitTransaction: function (tx, callbackOk, callbackError) {
      var $app = this

      console.log(tx)

      $app.ripple.submit(tx.txSigned.signedTransaction).then(function (txResponseData) {
        console.log(txResponseData)
        if (txResponseData.resultCode === 'tefPAST_SEQ') {
          swal('Transaction (error)', txResponseData.resultMessage, 'error')
        } else {
          if (tx.txSigned.id) $app.txQueue.push(tx)

          console.log('   >> [Tentative] Result: ', txResponseData.resultCode)
          console.log('   >> [Tentative] Message: ', txResponseData.resultMessage)
          callbackOk(txResponseData)
        }
      }).catch(function (e) {
        if (e.message === 'noNetwork') {
          // Retry @ s2
          $app.rippleFullHistory.submit(tx.txSigned.signedTransaction).then(function (txResponseData) {
            console.log(txResponseData)
            console.log('   >> [Tentative] Result: ', txResponseData.resultCode)
            console.log('   >> [Tentative] Message: ', txResponseData.resultMessage)
            callbackOk(txResponseData)
          }).catch(function (e) {
            console.log(e)
            callbackError(e)
          })
        } else {
          console.log(e)
          callbackError(e)
        }
      })
    },
    onConnected: function (f) {
      if (this.connected) {
        f()
      } else {
        this.awaitConnected.push(f)
      }
    },
    composeTx: function (tx) {
      this.$emit('compose', tx)
    },
    connectFullHistoryNode: function () {
      var $app = this
      this.rippleFullHistory = new ripple.RippleAPI({ server: 'wss://s2.ripple.com' })
      $app.rippleFullHistory.connect().then(() => {
        $app.rippleFullHistory.getServerInfo().then(function (server) {
          $app.rippleFullHistory.on('ledger', ledger => {
            if ($app.lastLedgerVersion === null || ledger.ledgerVersion > $app.lastLedgerVersion) $app.lastLedgerVersion = ledger.ledgerVersion
          })

          $app.rippleFullHistory.__serverInfo = server

          if ($app.lastLedgerVersion === null || server.validatedLedger.ledgerVersion > $app.lastLedgerVersion) $app.lastLedgerVersion = server.validatedLedger.ledgerVersion

          $app.connected = true
          console.log('S2 connected.')
          if ($app.awaitConnected.length > 0) {
            $app.awaitConnected.forEach(function (f) {
              f()
            })
          }
        })
      }).then(() => {
      }).catch(function (e) {
        console.log(e)
        $app.offline = true
        $app.$emit('nofallbacknetwork', e)
      })
    },
    connectedRipple: function (ripple) {
      var $app = this

      if (this.ripple === null) {
        this.ripple = ripple
        this.ripple.on('ledger', ledger => {
          $app.lastLedger = ledger
          $app.ledgerClosed(ledger)
        })

        if (window.eventBus.lastLedgerVersion === null || ripple.__serverInfo.validatedLedger.ledgerVersion > window.eventBus.lastLedgerVersion) window.eventBus.lastLedgerVersion = ripple.__serverInfo.validatedLedger.ledgerVersion

        $app.fee = parseFloat(ripple.__serverInfo.validatedLedger.baseFeeXRP).toFixed(6)
        console.log('Server base fee: ', $app.fee)
        if ($app.fee > $app.maxFee) $app.fee = $app.maxFee

        $app.startFeePoller($app)
      }
    },
    startFeePoller: function ($app) {
      setInterval(function () {
        if (!$app.offline) {
          $app.ripple.getFee().then(function (e) {
            var _fee = parseFloat(e).toFixed(6)
            if (_fee !== $app.fee) {
              $app.fee = _fee
              console.log('-- New estimated fee: ', $app.fee)
              if ($app.fee > $app.maxFee) $app.fee = $app.maxFee
              $app.$emit('fee', $app.fee)
            }
          })
        }
      }, 5000)
    },
    ledgerClosed: function (ledger) {
      this.$emit('ledger', ledger)
    },
    transactionReceived: function (message) {
      console.log('<< RX TX >>', message)
      this.$emit('tx', message)
    },
    subscribeAccount: function (account) {
      var $app = this

      if (this.connected) {
        if (this.subscribedAccounts.indexOf(account) < 0) {
          this.subscribedAccounts.push(account)
          console.log('Subscribe @ wallet ', account)
          var rippleConnection = $app.ripple && $app.ripple.connection ? $app.ripple.connection : $app.rippleFullHistory.connection
          rippleConnection.request({ command: 'subscribe', accounts: [ account ] })
          rippleConnection._ws.on('message', function (m) {
            var message = JSON.parse(m)
            if (message.type === 'transaction' && message.engine_result === 'tesSUCCESS') {
              // console.log(m)
              $app.transactionReceived(message)
            }
          })
        }
      } else {
        this.onConnected(function () {
          $app.subscribeAccount(account)
        })
      }
    }
  }
})

export default {
  name: 'App',
  data () {
    return {
      // rippledServer: 'rippled.xrptipbot.com',
      rippledServer: 's1.ripple.com',
      ripple: null,
      fee: null,
      wallets: null,
      connected: false,
      offline: false,
      switchedFullHistoryMode: false
    }
  },
  components: {
    Main
  },
  computed: {
    isConnected: function () {
      return this.connected && window.eventBus.connected
    }
  },
  watch: {
  },
  mounted: function () {
    var $app = this

    // Make getBalacnes public;
    window.getBalances = this.getBalances

    var appStartedOnline = true
    var initWhenOnline = function () {
      $app.ripple = new ripple.RippleAPI({ server: 'wss://' + $app.rippledServer })
      $app.connect()

      window.eventBus.$on('nofallbacknetwork', function (error) {
        if (!$app.connected) {
          // Offline mode, both rippled-servers: no connection
          console.log('Switching app to offline.')
          $app.offline = true
          $app.retrieveWallets()
        }
        return error
      })

      window.eventBus.$on('tx', function (message) {
        // console.log('ontx')
        console.log(message.transaction.Account + ' <> ' + message.transaction.Destination + ' @ ' + message.ledger_index, message.transaction)
        $app.getBalances()
      })
    }

    window.eventBus.offline = !navigator.onLine

    if (window.eventBus.offline) {
      $app.retrieveWallets()
      appStartedOnline = false
    } else {
      initWhenOnline()
    }

    window.addEventListener('online', function () {
      $app.offline = false
      window.eventBus.offline = false

      if (!appStartedOnline) {
        // Started offline, came online, init
        setTimeout(function () {
          window.eventBus.onConnected(function () {
            initWhenOnline()
            $app.getBalances()
          })
          window.eventBus.connectFullHistoryNode()
        }, 1500)
      }
    })

    window.addEventListener('offline', function () {
      $app.offline = true
      window.eventBus.offline = true
    })
  },
  methods: {
    removeWallet: function (wallet) {
      this.wallets.splice(this.wallets.indexOf(wallet), 1)
      this.persistWallets()
    },
    getBalances: function () {
      if (this.offline || window.eventBus.offline) {
        console.log('Skip getBalances, offline')
        return
      }

      var $app = this
      console.log('getBalances > connected? ', $app.connected)

      if (this.wallets !== null && this.wallets.length > 0) {
        this.wallets.forEach(function (w) {
          w.__fetchTransactions = function () {
            console.log('-- fetchtxs @ ' + w.address)
            window.eventBus.rippleFullHistory.getServerInfo().then(function (server) {
              var ledgers = server.completeLedgers.split('-')
              var firstLedger = parseInt(ledgers[0])
              var lastLedger = parseInt(ledgers.reverse()[0])
              window.eventBus.rippleFullHistory.getAccountInfo(w.address).then(function (accInfo) {
                Vue.set(w, '__sequence', parseInt(accInfo.sequence))

                try {
                  window.eventBus.rippleFullHistory.getTransactions(w.address, {
                    minLedgerVersion: firstLedger,
                    maxLedgerVersion: lastLedger,
                    types: [ 'payment' ],
                    limit: 100
                  }).then(function (r) {
                    console.log('<< txs [' + w.address + '] ', r)

                    Vue.set(w, '__tx', r)
                    Vue.set(w, '__txFetched', true)
                  })
                } catch (e) {
                  Vue.set(w, '__txFetched', e.message)
                  console.log(e)
                }
              }).catch(function (e) {
                Vue.set(w, '__txFetched', e.message)
                console.log(e)
              })
            })
          }

          window.eventBus.onConnected(function () {
            w.__fetchTransactions()
          })

          try {
            $app.ripple.getAccountInfo(w.address).then(function (accInfo) {
              Vue.set(w, 'balance', parseFloat(accInfo.xrpBalance))
              Vue.set(w, '__sequence', parseInt(accInfo.sequence))
            }).catch(function (e) {
              if (e.message === 'actNotFound') {
                Vue.set(w, 'balance', 0)
              } else
              if (e.message === 'noNetwork') {
                console.log('noNetwork, switching rippled?')
                $app.fallbackToS2()
                // TODO: check Endless Loop on no Network
                throw new Error('NO NETWORK, RETRY @ S2')
              } else {
                console.log(e)
              }
            })
          } catch (e) {
            console.log(e)
          }
        })
      }
    },
    retrieveWallets: function () {
      var $app = this
      localforage.getItem('wallets', function (err, r) {
        console.log('Retrieve wallets [err?] ', err)
        if (r === null || err) {
          r = []
        }
        if (r.length > 0) {
          r.forEach(function (w) {
            Vue.set(w, 'balance', null)
            Vue.set(w, '__tx', [])
            Vue.set(w, '__txFetched', false)
            Vue.set(w, '__sequence', 0)

            window.eventBus.subscribeAccount(w.address)
          })
        }
        $app.wallets = r
        $app.getBalances()
      })
    },
    addWallet: function (newWallet) {
      var existingWallet = this.wallets.filter(function (e) {
        return e.address === newWallet.address
      })
      if (existingWallet.length > 0) {
        if (existingWallet[0].key === '' && newWallet.key !== '') {
          this.wallets[this.wallets.indexOf(existingWallet[0])].key = newWallet.key
        }
      } else {
        Vue.set(newWallet, 'balance', null)
        Vue.set(newWallet, '__tx', [])
        Vue.set(newWallet, '__txFetched', false)
        Vue.set(newWallet, '__sequence', 0)

        this.wallets.push(newWallet)

        this.getBalances()
        this.persistWallets()

        window.eventBus.subscribeAccount(newWallet.address)
      }
    },
    persistWallets: function () {
      localforage.setItem('wallets', this.wallets.map(function (a) {
        var b = {}
        Object.keys(a).forEach(function (k) {
          if (k.substring(0, 2) !== '__' && k !== 'balance') {
            b[k] = a[k]
          }
        })
        return b
      }), function (err) {
        console.log('Store wallets [err?] ', err)
      })
    },
    fallbackToS2: function () {
      var $app = this
      if (!$app.switchedFullHistoryMode) {
        $app.ripple = window.eventBus.rippleFullHistory
        window.eventBus.onConnected(function () {
          $app.retrieveWallets()
          $app.connected = true
          // Fallback: poll for fee @ s2
          window.eventBus.startFeePoller($app)
        })
        console.info('[Rippled] Switched to fullHistory node.')
        $app.switchedFullHistoryMode = true
      }
    },
    connect: function () {
      var $app = this

      $app.ripple.connect().then(() => {
        $app.ripple.getServerInfo().then(function (server) {
          $app.ripple.__serverInfo = server
          console.log('Rippled connected.', server)
          $app.connected = true

          $app.retrieveWallets()

          window.eventBus.connectedRipple($app.ripple)

          // window.eventBus.subscribeAccount('rDsbeomae4FXwgQTJp9Rs64Qg9vDiTCdBv') // Bitstamp
          // window.eventBus.subscribeAccount('rLHzPsX6oXkzU2qL12kHCH8G8cnZv1rBJh') // Kraken
        })
      }).then(() => {
      }).catch(function (r) {
        console.error('Catch: connect:ripple.connect() ', r)
        $app.fallbackToS2()
      })
    }
  }
}
</script>

<style lang="scss">
  #app {
    font-family: Helvetica, Arial, sans-serif;
    font-size: .95rem;
    padding-top: 15px;
    padding-bottom: 15px;
    // -webkit-font-smoothing: antialiased;
    // -moz-osx-font-smoothing: grayscale;

    img#logo {
      max-width: 30vw;
      float: right;
      max-height: 2.5rem;
    }
  }
  a, button {
    cursor: pointer;
    &[disabled] {
      cursor: default;
    }
  }
  .swal-text {
    text-align: center;
  }
</style>
