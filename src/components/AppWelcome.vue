<template>
  <div class="main">
    <h5><b>Welcome</b></h5>

    <div v-if="!offline">
      <p>
        To start using kyte, you have to add a wallet first.
      </p>
      <p>
        You can either add a read-only wallet using a <b>public key (wallet address)</b> or
        a hot wallet using a <b>private key (secret)</b> or mnemonic password.
      </p>
      <p class="alert alert-warning">
          <i class="fa fa-info-circle"></i> When adding a hot wallet using the
          private key (secret) you are able to send XRP to other wallets.
      </p>
      <p class="text-muted">
        <i class="fa fa-exclamation-triangle"></i> If you enter a private key (secret)
        you will be asked to enter a passphrase. The passphrase (a sentence of 5+ words)
        will be used to encrypt your private key.
      </p>

      <button @click="next()" class="btn btn-block btn-primary btn-lg"><i class="fa fa-arrow-right"></i> Next (add wallet)</button>
      <!-- <div class="text-center text-muted">
        <small>- or -</small>
      </div>
      <button @click="composeBlank()" class="btn btn-block btn-outline-primary btn-s,"><i class="fa fa-arrow-right"></i> Compose transaction</button> -->

      <br />

      <div class="text-center text-muted">
        <!-- <small>Go offline to create an air-gapped transaction.</small> -->
        <small>Offline Air-gapped transactions: work in progress :)</small>
      </div>
    </div><!-- not offline -->

    <div v-if="offline && !composingTx">
      <p>
        You are using kyte in offline mode; no rippled-server could be reached.
      </p>
      <p>
        When using kyte in offline mode you can only create air-gapped transactions, meaning you
        can create and sign a transaction, and use an online computer to submit the transaction.
        This way, your private key will not be entered online <i class="fa fa-smile-o"></i>
      </p>
      <button @click="composeBlank()" class="btn btn-block btn-primary btn-md"><i class="fa fa-plus-circle"></i> Compose (air-gapped) transaction</button>
    </div><!-- offline -->
  </div>
</template>

<script>

export default {
  name: 'AppWelcome',
  props: [ 'next' ],
  components: {
  },
  data () {
    return {
    }
  },
  methods: {
    composeBlank: function () {
      this.$parent.composeBlank()
    }
  },
  computed: {
    offline: function () {
      return window.eventBus.offline
    },
    composingTx: function () {
      return this.$parent.composeTx !== null
    }
  },
  mounted: function () {
  }
}
</script>

<style lang="scss" scoped>
</style>
