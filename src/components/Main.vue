<template>
  <div class="main">
    <h2>XRP Wallet</h2>

    <div v-if="!submitAirGapped">
      <div v-if="offline" class="clearfix"></div>
      <p v-if="offline" class="alert alert-danger text-center"><i class="fa fa-power-off"></i> <b>Offline</b></p>

      <AppWelcome v-if="wallets.length < 1 && !addingWallet && !composeTx" :next="addWallet" />
      <AddWallet v-if="addingWallet" :next="addWalletDone" :cancel="cancel" :ripple="ripple" />

      <div v-if="!addingWallet && wallets.length > 0 && composeTx === null">
        <div v-if="walletDetails < 0">
          <WalletOverview :ripple="ripple" :wallets="wallets" :open="openWallet" :exchange-rate="exchangeRate" />
          <br />
          <button v-if="!offline" @click="addingWallet=true" class="btn btn-block btn-primary btn-md"><i class="fa fa-plus"></i> Add wallet</button>

          <button v-if="offline" @click="composeBlank()" class="btn btn-block btn-primary btn-md"><i class="fa fa-plus-circle"></i> Compose air gapped transaction</button>
          <button v-if="!offline" @click="submitAirGapped=true" class="btn btn-block btn-outline-primary btn-md"><i class="fa fa-send"></i> Send air gapped transaction</button>

          <div class="settingspane-to-be-module" v-if="!offline">
            <hr />

            <div class="card border-secondary text-secondary">
              <div class="card-block" style="padding: 10px 10px; padding-bottom: 0;">
                <h6><b>Settings</b></h6>
                <div class="form-group">
                  <label for="currency"><small>Currency</small></label>
                  <select v-model="currency" class="form-control form-control-sm" id="currency">
                    <option>EUR</option>
                    <option>USD</option>
                  </select>
                </div>
              </div>
            </div>
          </div><!-- settingspane -->

        </div>
        <div v-if="walletDetails > -1">
          <WalletDetails :ripple="ripple" :wallet="wallets[walletDetails]" :close="closeWallet" :persist-wallets="$parent.persistWallets" :exchange-rate="exchangeRate" />
        </div>
      </div>

      <div v-if="composeTx !== null">
        <TxCompose :tx="composeTx" :ripple="ripple" :wallets="wallets" :close="cancel" />
      </div>
    </div><!-- !submitAirGapped -->

    <div v-if="submitAirGapped && !offline">
      <h5><b>Submit air gapped transaction</b></h5>

      <p>
        Paste your offline generated transaction. Please check
        your wallet to see if your transaction is applied in
        a ledger.
      </p>

      <QrReader v-if="qr.scanning" :close="qrClose" :scan="qrScan" :disable="qrDisable">
        <p class="text-muted">
          Scan a Kyte air gapped transaction QR code, or a QR containing just a XRP transaction hash.
        </p>
      </QrReader>

      <button v-if="!qr.disabled" class="btn btn-block btn-outline-secondary" :disabled="qr.scanning" @click="qr.scanning=true"><i class="fa fa-qrcode"></i> Scan QR</button>
      <br />

      <textarea v-model="airGappedTx" id="airGappedTx" rows="11" class="form-control form-control-sm"></textarea>

      <div class="row">
        <div class="col">
          <button @click="submitAirGapped=false;airGappedTx=''" class="btn btn-block btn-outline-danger">
            <i class="fa fa-arrow-left"></i>
            Close
          </button>
        </div>
        <div class="col">
          <button @click="sendAirGappedTx" class="btn btn-block btn-primary" :disabled="!airGappedTxSanitized">
            <i class="fa fa-send"></i>
            Submit
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Vue from 'vue'
import QrReader from './QrReader'
import AppWelcome from './AppWelcome'
import AddWallet from './AddWallet'
import WalletOverview from './WalletOverview'
import WalletDetails from './WalletDetails'
import TxCompose from './TxCompose'
import swal from 'sweetalert'

Vue.filter('xrp', function (value, forcedDecimals) {
  var decimals = {
    min: typeof forcedDecimals === 'undefined' ? 0 : forcedDecimals,
    max: typeof forcedDecimals === 'undefined' ? 8 : forcedDecimals
  }
  return new window.Intl.NumberFormat(navigator.language, { style: 'decimal', minimumFractionDigits: decimals.min, maximumFractionDigits: decimals.max }).format(value)
})
Vue.filter('currency', function (value) {
  return new window.Intl.NumberFormat(navigator.language, { style: 'currency', currency: window.eventBus.currency }).format(value)
})

export default {
  name: 'Main',
  props: [ 'wallets', 'ripple' ],
  components: {
    AppWelcome,
    AddWallet,
    WalletOverview,
    WalletDetails,
    TxCompose,
    QrReader
  },
  data () {
    return {
      addingWallet: false,
      newWallet: {},
      walletDetails: -1,
      exchangeRate: { bid: 0 },
      composeTx: null,
      currency: '',
      submitAirGapped: false,
      airGappedTx: '',
      qr: {
        disabled: false,
        scanning: false
      }
    }
  },
  methods: {
    qrDisable: function () { this.qr.disabled = true },
    qrClose: function () { this.qr.scanning = false },
    qrScan: function (e) {
      if (e.trim().match(/^ripple:tx|^[0-9A-F]+$/i)) {
        this.qr.scanning = false
        this.airGappedTx = e
      } else {
        swal('Invalid transaction', 'You didn\'t scan a valid air gapped, signed transaction. Please use a QR code containing a raw hexadecimal transaction, or a transaction generated by Kyte.', 'error')
      }
    },
    sendAirGappedTx: function () {
      var $app = this

      var txId = ''
      var txSignedStr = this.airGappedTxSanitized
      var txMatch = this.airGappedTx.trim().toUpperCase().match(/^RIPPLE:TX:([A-F0-9]{10,}):([A-F0-9]{30,})$/)
      if (txMatch) {
        txId = txMatch[1]
      }

      window.eventBus.submitTransaction({
        txSigned: {
          id: txId,
          signedTransaction: txSignedStr
        }
      }, function (txResponseData) {
        // ok
        // $app.submitAirGapped = false
        // $app.airGappedTx = ''
        swal('Transaction sent', 'Tentative transaction: please wait for the transaction to be included in a closed ledger.', 'info')
        $app.submitAirGapped = false
        $app.airGappedTx = ''
      }, function (e) {
        // error
        swal('Transaction error', e.message, 'error')
      })
    },
    getExchangeRate: function () {
      if (!this.offline) {
        console.log('Update Exchange rate')
        var $app = this
        fetch('https://cors-anywhere.herokuapp.com/https://www.bitstamp.net/api/v2/ticker/xrp' + (window.eventBus.currency.toLowerCase()) + '/').then(function (e) {
          return e.json()
        }).then(function (e) {
          console.log('Exchange rate updated, ', e)
          $app.exchangeRate = e
        })
      }
    },
    cancel: function () {
      this.addingWallet = false
      this.composeTx = null
    },
    addWallet: function () {
      this.addingWallet = true
    },
    addWalletDone: function (newWallet) {
      this.newWallet = newWallet
      this.addingWallet = false
      this.$parent.addWallet(newWallet)
    },
    openWallet: function (wallet) {
      this.walletDetails = this.wallets.indexOf(wallet)
    },
    closeWallet: function () {
      this.walletDetails = -1
    },
    removeWallet: function (wallet) {
      this.$parent.removeWallet(wallet)
    },
    // setCurrency: function (e) {
    // console.log(e)
    // },
    composeBlank: function (c) {
      if (typeof c === 'undefined') c = {}
      this.composeTx = {
        from: c.from || '',
        to: c.to || '',
        tag: c.tag || '',
        amount: 0,
        tx: null,
        sent: false
      }
    }
  },
  computed: {
    offline: function () {
      return window.eventBus.offline
    },
    airGappedTxSanitized: function () {
      var txStr = this.airGappedTx.trim().toUpperCase()

      var txMatch = txStr.match(/^RIPPLE:TX:([A-F0-9]{10,}):([A-F0-9]{30,})$/)
      if (txMatch) {
        txStr = txMatch[2]
      }

      if (txStr.match(/^[A-F0-9]{30,}$/)) {
        return txStr
      }
      return ''
    }
  },
  watch: {
    currency: function (a) {
      window.eventBus.currency = a
    }
  },
  mounted: function () {
    var $app = this

    this.currency = window.eventBus.currency
    this.getExchangeRate()

    window.addEventListener('online', function () {
      $app.getExchangeRate()
    })

    clearInterval(window.getExchangeRate)
    window.getExchangeRate = setInterval(this.getExchangeRate, 60 * 1000)
    window.eventBus.$on('compose', function (c) {
      $app.composeBlank(c)
    })

    window.eventBus.$on('currencyswitch', function (c) {
      $app.exchangeRate = { bid: 0 }
      $app.getExchangeRate()
    })
  }
}
</script>

<style lang="scss" scoped>
  h2 {
    color: #2071EE;
    font-weight: bold;
  }
  textarea#airGappedTx {
    font-family: monospace;
    font-size: 11px;
    line-height: 12px;
    resize: none;
    margin-bottom: 8px;
  }
</style>
