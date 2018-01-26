<template>
  <div class="main">
    <h5><b>Compose Transaction</b></h5>

    <div v-if="qrMode">
      <p>
        You can either scan a <b>public key (wallet address)</b> or
        a Ripple URI.
      </p>

      <div v-show="!qrInitDone" class="text-center">
        <h1 class="text-muted"><i class="fa fa-circle-o-notch fa-spin"></i></h1>
      </div>
      <div class="qr-code-container" v-show="qrInitDone">
        <div class="qr-code-padding">
          <qrcode-reader @decode="onQrDecode" @init="onQrInit">
            <div class="crosshair"></div>
          </qrcode-reader>
        </div>
      </div>
      <br />
      <div class="text-center">
        <button @click="qrMode=false" class="btn btn-outline-danger"><i class="fa fa-times"></i> Cancel</button>
      </div>
    </div>

    <!-- {{ tx }} -->

    <div v-if="!qrMode">
      <div class="form-group">
        <label for="from">From XRP wallet</label>
        <select :disabled="submittingTx" v-model="tx.from" class="form-control" id="from" v-if="walletsWithTx.length > 0">
          <option :value="w.address" v-for="w in walletsWithTx" v-bind:key="w.address">{{ w.title }} ({{ w.balance|xrp }} XRP)</option>
          <option value="" v-if="offline">Other wallet (enter private key / secret / mnemonic)</option>
        </select>
      </div>

      <div class="form-group" v-if="(offline && walletsWithTx.length > 0 && tx.from === '') || walletsWithTx.length < 1">
        <label for="fromsecret">Sending Wallet: <b>Secret</b></label>
        <div class="input-group other-wallet">
          <input placeholder="Sending wallet (Private key / Secret / Mnemonic)" autocomplete="off" autocorrect="off" autocapitalize="off" spellcheck="false" autofocus v-model="tx.from" type="text" class="form-control form-control-sm" :class="{ 'is-invalid': !validFromAddress && tx.from !== '' }" id="fromsecret">
          <div class="input-group-append" v-if="!qrUnavailable">
            <button @click="qrMode=true;qrInitiator='from'" class="btn btn-outline-secondary" type="button"><i class="fa fa-qrcode"></i></button>
          </div>
        </div>

        <div class="card text-muted" v-if="(offline && walletsWithTx.length > 0 && tx.from === '') || walletsWithTx.length < 1">
          <small>
            Development: scanned secret
            {{ from }}
          </small>
        </div>
      </div>

      <div class="form-group" v-if="(!offline || tx.from !== '') && walletsWithTx.length > 0">
        <label for="passphrase">Passphrase</label>
        <Passphrase v-model="passphrase" :disabled="submittingTx" />
      </div>

      <p v-if="validPassphrase" class="alert alert-success text-center" style="padding-top: 5px; padding-bottom: 5px; margin-top: -10px;">
        <i class="fa fa-check"></i> <b>Valid passphrase</b>
      </p>
      <p v-if="passphrase && !validPassphrase" class="alert alert-warning text-center" style="padding-top: 5px; padding-bottom: 5px; margin-top: -10px;">
        <i class="fa fa-exclamation-triangle"></i> <b>Invalid passphrase</b>
      </p>

      <div class="form-group">
        <label for="to">To XRP wallet</label>
        <div class="input-group">
          <input :disabled="submittingTx" placeholder="Receiving wallet (rXXX...)" autocomplete="off" autocorrect="off" autocapitalize="off" spellcheck="false" autofocus v-model="tx.to" type="text" class="form-control form-control-sm" :class="{ 'is-invalid': !validToAddress && tx.to !== '' }" id="to">
          <div class="input-group-append" v-if="!qrUnavailable">
            <button :disabled="submittingTx" @click="qrMode=true;qrInitiator='to'" class="btn btn-outline-secondary" type="button"><i class="fa fa-qrcode"></i></button>
          </div>
        </div>
      </div>
      <div class="form-group">
        <label for="tag">Destination Tag</label>
        <input :disabled="submittingTx" autocomplete="off" autocorrect="off" autocapitalize="off" spellcheck="false" v-model="tx.tag" type="text" class="form-control form-control-sm" id="tag" :class="{ 'is-invalid': parseInt(tx.tag)+'' !== tx.tag+'' && tx.tag !== '' }">
        <small id="toHelp" class="form-text text-muted"><i class="fa fa-exclamation-triangle"></i> Probably required when sending XRP to an exchange!</small>
      </div>
      <div class="form-group">
        <div class="pull-right text-right" v-if="selectedWallet.available > 0">
          <small class="text-muted">
            <i class="fa fa-info-circle"></i>
            Available: <b class="text-primary">{{ selectedWallet.available|xrp }} XRP</b>
          </small>
        </div>
        <label for="amount">
          <b>Amount (XRP)</b>
        </label>
        <div class="input-group">
          <input :disabled="submittingTx" autocomplete="off" autocorrect="off" autocapitalize="off" spellcheck="false" min="0" step="0.1" v-model.number="tx.amount" type="number" class="form-control form-control-lg text-center" :class="{ 'is-invalid': tx.amount > 0 && !validAmount }" id="amount">
          <div class="input-group-append" v-if="tx.amount >= 0 && exchangeRate.bid && !offline">
            <span style="min-width: 120px;" class="input-group-text text-muted">
              <span style="margin: 0 auto 0 auto;">{{ (exchangeRate.bid * tx.amount) | currency }}</span>
            </span>
          </div>
        </div>
      </div>

      <div class="form-group alert alert-warning" v-if="txAdvanced.show">
        <h6 class=""><b>Advanced</b></h6>
        <div class="row">
          <div class="col-6">
            <a v-if="txAdvanced.fee !== mostRecentFee" @click="txAdvanced.fee=mostRecentFee" class="pull-right">
              <i class="fa fa-undo"></i>
              <u>Reset</u>
            </a>
            <label for="fee">
              <small><b>Transaction fee</b></small>
            </label>
            <div class="input-group">
              <input autocomplete="off" autocorrect="off" autocapitalize="off" spellcheck="false" min="0.00001" step="0.000001" v-model.number="txAdvanced.fee" type="number" class="form-control form-control-sm text-center" id="fee">
              <div class="input-group-append">
                <span class="input-group-text" style="padding-left: 4px; padding-right: 4px; font-size: 0.8rem;">XRP</span>
              </div>
            </div>
          </div>
          <div class="col-6 text-right">
            <label class="" for="ledgers"><small><b>Ledger Close Timeout</b></small></label>
            <input :disabled="offline" autocomplete="off" autocorrect="off" autocapitalize="off" spellcheck="false" min="5" max="20" step="1" v-model.number="txAdvanced.ledgerClose" type="number" class="form-control form-control-sm text-center" id="ledgers">
          </div>
        </div>
      </div>

      <div class="row buttons">
        <div class="col col-4-sm">
          <button v-if="!txAdvanced.show" @click="close" class="btn btn-outline-danger btn-block" :disabled="submittingTx"><i class="fa fa-arrow-left"></i> Cancel</button>
        </div>
        <div class="col col-4-sm">
          <button v-if="!txAdvanced.show" @click="txAdvanced.show=true" class="btn btn-block btn-outline-secondary" :disabled="submittingTx"><i class="fa fa-asterisk"></i> Advanced</button>
          <button v-if="txAdvanced.show" @click="txAdvanced.show=false" class="btn btn-block btn-outline-primary"><i class="fa fa-arrow-left"></i> Back</button>
        </div>
        <div class="col col-4-sm">
          <button @click="signTransaction()" v-if="!txAdvanced.show" class="btn btn-block" :class="{ 'btn-primary' : allowSending, 'btn-outline-primary' : !allowSending, 'disabled' : !allowSending }" :disabled="!allowSending || submittingTx">
            <span v-if="!submittingTx && !offline"><i class="fa fa-paper-plane"></i> Submit</span>
            <span v-if="!submittingTx && offline"><i class="fa fa-cogs"></i> Create</span>
            <span v-if="submittingTx"><i class="fa fa-refresh fa-spin"></i></span>
          </button>
        </div>
      </div>
    </div><!-- qrMode -->
  </div>
</template>

<script>
import { QrcodeReader } from 'vue-qrcode-reader'
import Passphrase from './Passphrase'
import swal from 'sweetalert'
import CryptoJS from 'crypto-js'

export default {
  name: 'TxCompose',
  props: [ 'tx', 'wallets', 'ripple', 'close' ],
  components: {
    QrcodeReader,
    Passphrase
  },
  data () {
    return {
      qrMode: false,
      qrInitDone: false,
      qrUnavailable: false,
      passphrase: '',
      qrInitiator: 'to',
      from: {
        address: '',
        secret: ''
      },
      txAdvanced: {
        show: false,
        fee: window.eventBus.fee,
        ledgerClose: 5
      },
      mostRecentFee: window.eventBus.fee,
      submittingTx: false
    }
  },
  watch: {
  },
  methods: {
    signTransaction: function () {
      var $app = this

      // Todo: sign with custom from address

      setTimeout(function () {
        $app.submittingTx = false
      }, 3000)

      var tx = {
        TransactionType: 'Payment',
        Account: this.tx.from,
        Fee: (this.txAdvanced.fee * 1000 * 1000).toFixed(0) + '',
        Destination: this.tx.to,
        DestinationTag: this.tx.tag || null,
        Amount: (this.tx.amount * 1000 * 1000).toFixed(0) + ''
      }

      var signTx = function () {
        console.log('tx', tx)
        var jsonTx = JSON.stringify(tx)
        var signedTx = {}

        if ($app.validPassphrase.wallet.hexAddress && $app.validPassphrase.wallet.hexAddress !== '') {
          // Hex key
          const sign = require('ripple-sign-keypairs')
          signedTx = sign(jsonTx, {
            publicKey: $app.validPassphrase.wallet.hexAddress,
            privateKey: $app.validPassphrase.secret
          })
        } else {
          // String key
          console.log($app.validPassphrase.secret)
          signedTx = $app.ripple.sign(jsonTx, $app.validPassphrase.secret)
        }

        window.eventBus.submitTransaction({
          from: $app.validPassphrase.wallet.address,
          txJson: tx,
          txSigned: signedTx
        }, function (txResponseData) {
          $app.submittingTx = false
          $app.close() // Return to the wallet
          swal('Transaction sent', 'Tentative transaction: please wait for the transaction to either be included in a closed ledger or timeout.', 'info')
        }, function (e) {
          $app.submittingTx = false
          swal('Transaction error', e.message, 'error')
        })
      }

      if (!this.offline) {
        tx.LastLedgerSequence = window.eventBus.lastLedgerVersion + $app.txAdvanced.ledgerClose

        try {
          $app.ripple.getAccountInfo(this.tx.from).then(function (accInfo) {
            tx.Sequence = accInfo.sequence
            signTx()
          }).catch(function (e) {
            swal('Error getting account info', e.message, 'error')
            $app.submittingTx = false
          })
        } catch (e) {
          swal('Error', e.message, 'error')
          $app.submittingTx = false
        }
      } else {
        signTx()
      }
    },
    decryptPassphrase: function (text, secret) {
      try {
        var key = CryptoJS.PBKDF2(secret, CryptoJS.enc.Hex.parse(text.substr(0, 32)) /* Salt */, { keySize: 256 / 32, iterations: 100 })
        var decrypted = CryptoJS.AES.decrypt(text.substring(64) /* encrypted */, key, { iv: CryptoJS.enc.Hex.parse(text.substr(32, 32)) /* iv */, padding: CryptoJS.pad.Pkcs7, mode: CryptoJS.mode.CBC })
        return decrypted.toString(CryptoJS.enc.Utf8)
      } catch (e) {
        return null
      }
    },
    onQrDecode: function (e) {
      this.qrMode = false
      console.log('QR result [' + this.qrInitiator + '] ', e)

      if (this.qrInitiator === 'from') {
        // Check for private, multiple possible
        swal('Oops', 'Invalid value: please scan a private key (secret), hexadecimal private key or mnemonic phrase', 'error')
      }
      if (this.qrInitiator === 'to') {
        // Check for valid wallet address ^rXXX
        //  - rXXXX
        //  - Ripple URI (+ possible dtag)
        //  - Ripple: rXXXXX
        if (e.match(/^r[a-zA-Z0-9]+$/)) {
          this.tx.to = e
        } else if (e.match(/^Ripple: r[a-zA-Z0-9]+/)) {
          this.tx.to = e.split(':')[1].trim()
        } else if (e.match(/https.+ripple\.com.+send\?/)) {
          var toMatch = e.match(/[?&]to=(r[a-zA-Z0-9]+)/)
          if (toMatch) {
            this.tx.to = toMatch[1]
            var toDtag = e.match(/[?&]dt=([0-9]+)/)
            if (toDtag) {
              this.tx.tag = parseInt(toDtag[1])
            }
            var toAmount = e.match(/[?&]amount=([0-9.]+)/)
            if (toAmount) {
              this.tx.amount = parseFloat(toAmount[1])
            }
          } else {
            swal('Oops', 'Could not parse destination address from Ripple URI', 'error')
          }
        } else {
          swal('Oops', 'Invalid destination: no XRP wallet address could be found', 'error')
        }
      }
    },
    async onQrInit (promise) {
      // show loading indicator
      try {
        await promise
        // successfully initialized
        this.qrInitDone = true
      } catch (error) {
        if (error.name === 'NotAllowedError') {
          swal('Oops', 'User denied camera access permisson', 'error')
        } else if (error.name === 'NotFoundError') {
          swal('Oops', 'No suitable camera device installed', 'error')
          this.qrUnavailable = true
        } else if (error.name === 'NotSupportedError') {
          swal('Oops', 'Page is not served over HTTPS (or localhost)', 'error')
          this.qrUnavailable = true
        } else if (error.name === 'NotReadableError') {
          swal('Oops', 'Maybe camera is already in use', 'error')
        } else if (error.name === 'OverconstrainedError') {
          swal('Oops', 'Passed constraints don\'t match any camera. Did you requested the front camera although there is none?', 'error')
          this.qrUnavailable = true
        } else {
          swal('Oops', 'Browser is probably lacking features (WebRTC, Canvas)', 'error')
          this.qrUnavailable = true
        }
        this.qrMode = false
      } finally {
        // hide loading indicator
      }
    }
  },
  computed: {
    validPassphrase: function () {
      var $app = this
      if (this.tx.from.match(/^r[a-zA-Z0-9]+$/)) {
        var wallet = null
        var wallets = this.walletsWithTx.filter(function (a) {
          return a.address === $app.tx.from
        })
        if (wallets.length > 0) {
          wallet = wallets[0]
        }
        var decryptedPassphrase = this.decryptPassphrase(wallet.key, $app.passphrase)
        if (decryptedPassphrase !== null && decryptedPassphrase.match(/^[0-9A-F]{64}$|^00[0-9A-F]{64}$|^s[a-zA-Z0-9]{10,}$/)) {
          return {
            wallet: wallet,
            secret: decryptedPassphrase
          }
        }
      }
      return false
    },
    walletsWithTx: function () {
      return this.wallets.filter(function (w) {
        return w.tx
      })
    },
    validAmount: function () {
      if (!this.offline) {
        if (this.tx.amount >= this.selectedWallet.available) {
          if (this.selectedWallet.address !== '') {
            return false
          }
        }
      }
      return this.tx.amount > 0
    },
    allowSending: function () {
      if (this.validFromAddress && this.validToAddress && this.validAmount && (this.validPassphrase || this.manualFromAddress)) {
        return true
      }
      return false
    },
    validToAddress: function () {
      if (this.tx.to.trim().match(/^r[a-zA-Z0-9]+$/)) {
        return true
      }
      return false
    },
    manualFromAddress: function () {
      return !this.tx.from.match(/^r[a-zA-Z0-9]+$/) && this.validFromAddress
    },
    validFromAddress: function () {
      // Wallet from list
      if (this.tx.from.match(/^r[a-zA-Z0-9]+$/)) {
        return true
      }
      // Entered values
      if (this.from.secret.trim().match(/^[s][a-zA-Z0-9]+$/)) {
        return true
      }
      if (this.from.secret.trim().match(/^[0-9A-F]{64}$|^00[0-9A-F]{64}$/)) {
        return true
      }
      if (this.from.secret.trim().match(/^[a-z]+ [a-z ]+ [a-z]+$/)) {
        return true
      }
      return false
    },
    selectedWallet: function () {
      var $app = this
      if (!this.validFromAddress || this.offline) return { address: '', balance: 0, available: 0 }
      return {
        address: this.tx.from,
        balance: this.walletsWithTx.filter(function (w) { return w.address === $app.tx.from })[0].balance,
        available: this.walletsWithTx.filter(function (w) { return w.address === $app.tx.from })[0].balance - this.ripple.__serverInfo.validatedLedger.reserveBaseXRP
      }
    },
    offline: function () {
      return window.eventBus.offline
    },
    exchangeRate: function () {
      return {
        bid: this.$parent.exchangeRate.bid,
        currency: window.eventBus.currency
      }
    }
  },
  mounted: function () {
    var $app = this
    window.eventBus.$on('fee', function (f) {
      var baseFee = parseFloat($app.ripple.__serverInfo.validatedLedger.baseFeeXRP)
      if ($app.mostRecentFee === $app.txAdvanced.fee && f > baseFee) {
        $app.txAdvanced.fee = f
      }
      $app.mostRecentFee = f
    })
  },
  filters: {
  }
}
</script>

<style lang="scss" scoped>
  div.main {
    padding-bottom: 20px;
  }

  div.buttons {
    div.col {
      button {
        margin-bottom: 15px;
      }
    }
  }

  input {
    &.form-control-lg {
      font-weight: bold;
    }
  }

  div.qr-code-container {
    border: 2px solid #ccc;
    border-radius: 10px;
    overflow: hidden;
    width: 90vw;
    height: 90vw;
    max-width: 500px;
    max-height: 500px;
    margin: 0 auto 0 auto;

    div.qr-code-padding {
      border: 2px solid white;
      width: 100%;
      height: 100%;
      border-radius: 6px;
      overflow: hidden;

      div.crosshair {
        border: 2px solid white;
        width: 60%;
        height: 60%;
        border-radius: 7px;
        position: absolute;
        transform: translate(-50%, -50%);
        left: 50%;
        top: 50%;
      }
    }
  }
</style>

<style lang="scss">
  .qrcode-reader {
    width: 100%;
    height: 100%;
    border-radius: 6px;
    overflow: hidden;
  }
  video.qrcode-reader__camera {
    width: 200%;
    height: 200%;
    max-width: 200%;
    max-height: 200%;
    position: absolute;
    transform: translate(-50%, -50%);
    left: 50%;
    top: 50%;
  }
</style>
