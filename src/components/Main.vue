<template>
  <div class="main">
    <h2>XRP Wallet</h2>

    <div v-if="offline" class="clearfix"></div>
    <p v-if="offline" class="alert alert-danger text-center"><i class="fa fa-power-off"></i> <b>Offline</b></p>

    <AppWelcome v-if="wallets.length < 1 && !addingWallet && !composeTx" :next="addWallet" />
    <AddWallet v-if="addingWallet" :next="addWalletDone" :cancel="cancel" :ripple="ripple" />

    <div v-if="!addingWallet && wallets.length > 0 && composeTx === null">
      <div v-if="walletDetails < 0">
        <WalletOverview :ripple="ripple" :wallets="wallets" :open="openWallet" :exchange-rate="exchangeRate" />
        <br />
        <button v-if="!offline" @click="addingWallet=true" class="btn btn-block btn-primary btn-md"><i class="fa fa-plus"></i> Add wallet</button>
        <!-- Todo: submit air gapped -->
        <!-- <button v-if="!offline" @click="airgappedTx=true" class="btn btn-block btn-outline-secondary btn-sm"><i class="fa fa-paper-plane"></i> Submit (air-gapped) transaction</button> -->

        <button v-if="offline" @click="composeBlank()" class="btn btn-block btn-primary btn-md"><i class="fa fa-plus-circle"></i> Compose (air-gapped) transaction</button>

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
  </div>
</template>

<script>
import Vue from 'vue'
import AppWelcome from './AppWelcome'
import AddWallet from './AddWallet'
import WalletOverview from './WalletOverview'
import WalletDetails from './WalletDetails'
import TxCompose from './TxCompose'

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
    TxCompose
  },
  data () {
    return {
      addingWallet: false,
      newWallet: {},
      walletDetails: -1,
      exchangeRate: { bid: 0 },
      composeTx: null,
      currency: ''
    }
  },
  methods: {
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
</style>
