<template>
  <div class="main">
    <div v-if="qrMode && !pickPassphrase">
      <h5><b>QR Scanner</b></h5>
      <p>
        You can either scan a <b>public key (wallet address)</b> to add
        a read-only wallet or scan a <b>private key (secret)</b> to add
        a hot wallet.
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

    <div v-if="!qrMode && !pickPassphrase">
      <label for="wallet">
        <h5><b>Enter your key or mnemonic</b></h5>
        <div class="text-muted">
          Your public key starts with a lowercase <b>r</b>, a private key either starts with
          a lowercase <b>s</b> or is a long code containing the digits 0-9 and letters A-F.
        </div>
      </label>
      <div class="input-group">
        <input v-model="walletKey" autocomplete="off" autocorrect="off" autocapitalize="off" spellcheck="false" id="wallet" name="wallet" class="form-control input form-control" :class="{ 'small-font': !qrUnavailable }" autofocus />
        <div class="input-group-append" v-if="!qrUnavailable">
          <button @click="qrMode=true" class="btn btn-outline-secondary" type="button"><i class="fa fa-qrcode"></i></button>
        </div>
      </div>

      <br />

      <div v-if="walletKeyDetails.valid">
        <small>
          <div v-if="walletKeyDetails.type === 'A'" class="text-success">
            <ul class="list-unstyled">
              <li><i class="fa fa-check fa-fw"></i> This looks like a <b>wallet address</b></li>
              <li class="text-danger"><i class="fa fa-times fa-fw"></i> You will <b>not</b> be able to submit transactions</li>
            </ul>
          </div>
          <div v-if="walletKeyDetails.type === 'F'" class="text-success">
            <ul class="list-unstyled">
              <li><i class="fa fa-check fa-fw"></i> This looks like a <b>secret (family seed)</b></li>
              <li><i class="fa fa-check fa-fw"></i> You will be able to send transactions</li>
            </ul>
          </div>
          <div v-if="walletKeyDetails.type === 'H'" class="text-success">
            <ul class="list-unstyled">
              <li><i class="fa fa-check fa-fw"></i> This looks like a <b>hexadecimal secret</b></li>
              <li><i class="fa fa-check fa-fw"></i> You will be able to send transactions</li>
            </ul>
          </div>
          <div v-if="walletKeyDetails.type === 'M'" class="text-success">
            <ul class="list-unstyled">
              <li><i class="fa fa-check fa-fw"></i> This looks like a <b>mnemonic password</b></li>
              <li><i class="fa fa-check fa-fw"></i> You will be able to send transactions</li>
            </ul>
          </div>
        </small>

        <button v-if="walletKeyDetails.tx" @click="pickPassphrase=true" class="btn btn-block btn-primary btn-lg"><i class="fa fa-arrow-right"></i> Next (enter passphrase)</button>
        <button v-if="!walletKeyDetails.tx && !addingWallet" @click="nextAddWallet()" class="btn btn-block btn-primary btn-lg"><i class="fa fa-arrow-right"></i> Finish</button>
      </div>

      <div v-show="!addingWallet">
        <br />
        <div class="text-center">
          <small>
            <a @click="cancel()" class="text-muted"><i class="fa fa-arrow-left"></i> <u>Cancel</u></a>
          </small>
        </div>
      </div>
    </div>

    <div v-if="pickPassphrase">
      <h5><b>Enter a passphrase</b></h5>
      <p>
        Enter a passphrase. Your passphrase should be a
        case insensitive sentence of <b>6 or more words</b>.
      </p>
      <p class="alert alert-warning">
        You will have to enter your passphrase to sign transactions if you want to send XRP
        to another wallet. Your passphrase is linked <b>to this specific wallet</b>.
      </p>

      <Passphrase v-model="passphrase" />
      <div class="repeat-passphrase" v-if="passphrase.length > 0">
        <Passphrase v-model="passphrase2" placeholder="Repeat your passphrase (just to be sure)" />
      </div>
      <br />
      <button v-if="passphrase.length > 0 && !addingWallet && passphrase !== '' && passphrase === passphrase2" @click="nextAddWallet()" class="btn btn-block btn-primary btn-lg"><i class="fa fa-arrow-right"></i> Finish</button>

      <div v-show="!addingWallet">
        <br /><br />
        <div class="text-center">
          <small>
            <a @click="pickPassphrase=false" class="text-muted"><i class="fa fa-arrow-left"></i> <u>Cancel</u></a>
          </small>
        </div>
      </div>
    </div>

    <div class="text-center" v-show="addingWallet">
      <h1 class="text-muted"><i class="fa fa-circle-o-notch fa-spin"></i></h1>
    </div>

  </div>
</template>

<script>
import swal from 'sweetalert'
import { QrcodeReader } from 'vue-qrcode-reader'
import Passphrase from './Passphrase'
import CryptoJS from 'crypto-js'

export default {
  name: 'AddWallet',
  props: [ 'cancel', 'next', 'ripple' ],
  components: {
    QrcodeReader,
    Passphrase
  },
  data () {
    return {
      qrMode: false,
      qrInitDone: false,
      qrUnavailable: false,
      walletKey: '',
      pickPassphrase: false,
      passphrase: '',
      passphrase2: '',
      addingWallet: false
    }
  },
  methods: {
    encrypt: function (text, secret) {
      var salt = CryptoJS.lib.WordArray.random(128 / 8)
      var iv = CryptoJS.lib.WordArray.random(128 / 8)
      var encrypted = CryptoJS.AES.encrypt(text, CryptoJS.PBKDF2(secret, salt, { keySize: 256 / 32, iterations: 100 }) /* key */, { iv: iv, padding: CryptoJS.pad.Pkcs7, mode: CryptoJS.mode.CBC })
      var transitmessage = salt.toString() + iv.toString() + encrypted.toString()
      return transitmessage
    },
    retry: function (e) {
      swal('Oops', e.message, 'error')
      this.pickPassphrase = false
      this.walletKey = ''
      this.addingWallet = false
    },
    nextAddWallet: function () {
      var $app = this
      this.addingWallet = true
      this.$nextTick(function () {
        setTimeout(function () {
          var address = ''
          var key = ''
          var hexAddress = ''

          if ($app.walletKeyDetails.type === 'A') {
            address = $app.walletKeyDetails.string
          }
          if ($app.walletKeyDetails.type === 'F') {
            const keypairs = require('ripple-keypairs')
            var secret = $app.walletKeyDetails.string
            var keypair = keypairs.deriveKeypair(secret)
            key = secret
            address = keypairs.deriveAddress(keypair.publicKey)
          }
          if ($app.walletKeyDetails.type === 'H') {
            const keypairs = require('ripple-keypairs')
            const elliptic = require('elliptic')
            const secp256k1 = elliptic.ec('secp256k1')

            var hexsecret = $app.walletKeyDetails.string
            if (hexsecret.length === 66) hexsecret = hexsecret.substring(2)
            key = hexsecret

            var bytesToHex = function (a) {
              return a.map(function (byteValue) {
                const hex = byteValue.toString(16).toUpperCase()
                return hex.length > 1 ? hex : '0' + hex
              }).join('')
            }

            var privateKey = '00' + hexsecret
            const publicKey = bytesToHex(secp256k1.keyFromPrivate(privateKey.slice(2)).getPublic().encodeCompressed())
            hexAddress = publicKey
            address = keypairs.deriveAddress(publicKey)
          }
          if ($app.walletKeyDetails.type === 'M') {
            const bip39 = require('bip39')
            const bip32 = require('ripple-bip32')
            const keypairs = require('ripple-keypairs')
            var mnemonic = $app.walletKeyDetails.string.toLowerCase().trim().replace(/ +/g, ' ')
            const seed = bip39.mnemonicToSeed(mnemonic)
            const m = bip32.fromSeedBuffer(seed)
            const keyPair = m.derivePath("m/44'/144'/0'/0/0").keyPair.getKeyPairs()
            key = keyPair.privateKey
            hexAddress = keyPair.publicKey
            address = keypairs.deriveAddress(keyPair.publicKey)
          }

          var wallet = {
            address: address,
            hexAddress: hexAddress,
            key: key.length > 0 ? $app.encrypt(key, $app.passphrase) : '',
            tx: $app.walletKeyDetails.tx,
            title: address
          }

          try {
            $app.ripple.getAccountInfo(wallet.address).then(function (accInfo) {
              $app.next(wallet)
            }).catch(function (e) {
              if (e.message === 'actNotFound') {
                $app.next(wallet)
              } else if (e.message === 'noNetwork') {
                $app.retry(e)
                console.log('noNetwork.....')
              } else {
                $app.retry(e)
              }
            })
          } catch (e) {
            $app.retry(e)
          }
        }, 250)
      })
    },
    onQrDecode: function (e) {
      if (e.match(/ripple:/i)) {
        this.walletKey = e.split(':')[1].trim()
      } else
      if (e.match(/^[rs0-9a-f][0-9a-zA-Z]+$/i)) {
        this.walletKey = e.split(' ')[0].trim()
      } else
      if (e.match(/to=/)) {
        this.walletKey = e.split('to=')[1].split('&')[0].trim()
      } else
      if (e.match(/^[a-z ]{10,}$/)) {
        this.walletKey = e.trim()
      }

      this.qrMode = false
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
    walletKeyDetails: function () {
      var type = 'X'
      var valid = false
      var tx = false
      var string = this.walletKey.trim()

      if (string.match(/^r[a-zA-Z0-9]{10,}/)) {
        type = 'A'
        valid = true
        tx = false
      }
      if (string.match(/^s[a-zA-Z0-9]{16,}/)) {
        type = 'F'
        valid = true
        tx = true
      }
      if (string.match(/^[a-fA-F0-9]{64}$|^00[a-fA-F0-9]{64}$/)) {
        type = 'H'
        valid = true
        tx = true
      }
      if (string.match(/^[a-z]{1,}[ a-z]{8,}[a-z]{1,}$/)) {
        type = 'M'
        valid = true
        tx = true
      }

      return { type: type, valid: valid, string: string, tx: tx }
    }
  },
  mounted: function () {
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss" scoped>
  h2 {
    color: #2071EE;
    font-weight: bold;
  }

  input#wallet {
    &.small-font {
      font-size: .8rem;
    }
  }

  .repeat-passphrase { margin-top: 10px; }

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
