<template>
  <div class="main">
    <h2>Welcome</h2>
    <p>
      To start using <b>kyte</b>, you have to configure a passphrase.
      This passphrase is used to encrypt your wallet secrets.
    </p>
    <p>
      <b class="text-danger">
      <i class="fa fa-exclamation-triangle"></i>
      It is very important you keep the passphrase safe!</b>
      If someone has access to this device and your passphrase, your
      XRP can be stolen.
    </p>
    <p>
      A passphrase is a (case insensitive) sentence containing a <b>minimum of 6 words</b>. Only words &gt; 2 characters will be counted.
    </p>
    <Passphrase v-model="passphrase" :length="5" :safelength="8" />
    <p class="text-muted">
      <br />
      <small>
        If you forget your passphrase, you will have to reinitialize <b>kyte</b>, and
        add your wallets again. <b>It is important to save the private keys, in case
        you want you want to use your wallets elswhere or if you forget your passphrase.</b>
      </small>
    </p>
    <button class="btn-block btn btn-lg" :class="{ 'btn-grey': passphrase.length < 1, 'btn-primary': passphrase.length > 0 }" :disabled="passphrase.length < 1">
      <i class="fa fa-arrow-right"></i> Next
    </button>
  </div>
</template>

<script>
import Passphrase from './Passphrase'

export default {
  name: 'PassphraseSample',
  components: {
    Passphrase
  },
  data () {
    return {
      passphrase: ''
    }
  },
  computed: {
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
  input.passphrase {
    padding-right: 50px;
  }
  span.passphrase-wordcount {
    padding: 0px 10px;
    font-weight: bold;
    border-radius: 4px;
    position: absolute;
    margin-top: 8px;
    right: 0;
    margin-right: 24px;
    color: white;
  }
</style>
