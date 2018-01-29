<template>
  <div class="main">
    <div v-if="txDetails!==null" id="exampleModalLive" class="modal fade show" tabindex="-1" role="dialog" style="display: block; background-color: rgba(0,0,0,.3);">
      <div class="modal-dialog" role="document">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">
              Transaction
              <span class="pull-left">
                <i class="fa fa-fw fa-paper-plane" v-if="txDetails.specification.source.address === wallet.address"></i>
                <i class="fa fa-fw fa-inbox" v-if="txDetails.specification.source.address !== wallet.address"></i>
              </span>
            </h5>
            <button type="button" class="close" @click="closeTxDetails" aria-label="Close">
              <span aria-hidden="true">×</span>
            </button>
          </div>
          <div class="modal-body">
            <TxDetails :tx="txDetails" :wallet="wallet" />
          </div>
          <div class="modal-footer">
            <button type="button" class="btn btn-secondary" @click="closeTxDetails">Close</button>
          </div>
        </div>
      </div>
    </div>

    <div v-if="walletSettings">
      <h5><b>Wallet settings</b></h5>

      <h6 class="text-center"><small><code>{{ wallet.address }}</code></small></h6>

      <label for="wallet">
        Title
      </label>
      <input v-model="wallet.title" autocomplete="off" autocorrect="off" autocapitalize="off" spellcheck="false" id="wallet" name="wallet" class="form-control input form-control" autofocus />
      <br />

      <div class="row">
        <div class="col-6"><button @click="walletSettings=false" class="btn btn-block btn-sm btn-outline-secondary"><i class="fa fa-arrow-left"></i> Cancel</button></div>
        <div class="col-6"><button @click="persistWallets();walletSettings=false" class="btn btn-block btn-sm btn-primary"><i class="fa fa-check"></i> Save</button></div>
      </div>
    </div>

    <div v-if="!walletSettings">
      <div class="card">
        <div class="card-header text-center" style="padding-top: 10px;">
          <a style="margin-left: -10px;" @click="close()" class="pull-left text-primary"><i class="fa fa-arrow-left"></i></a>
          <b v-if="wallet.address !== wallet.title">{{ wallet.title }}</b>
          <b v-if="wallet.address === wallet.title">XRP Wallet</b>
          <a style="margin-right: -10px;" @click="walletSettings=true" class="pull-right text-primary"><i class="fa fa-pencil"></i></a>
          <div class="row" style="margin-top: 10px; padding-top: 10px; border-top: 1px solid #cfcfcf;" v-if="!offline">
            <div class="col-6 text-left text-muted" v-if="exchangeRate.bid">{{ (wallet.balance * exchangeRate.bid) | currency }}</div>
            <div :class="{ 'col-6': exchangeRate.bid, 'text-right': exchangeRate.bid, 'text-center': !exchangeRate.bid, 'col-12' : !exchangeRate.bid }">
              <div class="text-center" v-if="wallet.balance === null">
                <i class="fa fa-spin fa-refresh"></i>
                <br />
                <small>Loading balance...</small>
              </div>
              <b class="text-primary" v-if="wallet.balance !== null">{{ wallet.balance | xrp }} XRP</b>
            </div>
          </div>
        </div>
        <div class="card-block" style="padding: 10px;">
          <h6 class="card-title text-center">
            <code>{{ wallet.address }}</code>
          </h6>
          <div class="text-center">
            <QrCode :value="wallet.address" :options="{ size: 150 }"></QrCode>
          </div>
          <button @click="composeTx()" v-if="!offline && wallet.key && wallet.balance > reserveXRP" style="margin-top: 5px;" class="btn btn-block btn-sm btn-primary"><i class="fa fa-plus-circle"></i> New transction</button>
          <button @click="composeTx()" v-if="offline || !wallet.key" style="margin-top: 5px;" class="btn btn-block btn-sm btn-primary"><i class="fa fa-plus-circle"></i> Create air gapped transaction</button>
        </div>
      </div>

      <br />

      <div v-if="tentativeTxs.length > 0">
        <!-- Todo: Tentative txs: don't show if already in TX history -->
        <h5><b>Tentative transaction(s)</b></h5>

        <p class="alert alert-warning text-center" style="padding-top: 2px; padding-bottom: 2px;">
          <small>
            <i class="fa fa-info-circle"></i>
            These transactions are still being processed.
          </small>
        </p>

        <small>
          <ul class="list-group">
            <li class="list-group-item" v-for="t in tentativeTxs" v-bind:key="t.txSigned.id">
              <code class="text-primary">
                <b>{{ t.txJson.Destination }}</b>
                <span v-if="t.txJson.DestinationTag" class="text-secondary"><i class="fa fa-tag"></i> {{ t.txJson.DestinationTag }}</span>
              </code>
              <!-- <br />
              <code>{{ t.txJson.LastLedgerSequence }}</code>
              <br />
              <code>{{ lastLedgerVersion }}</code> -->
              <!-- <span class="pull-right">
                &nbsp;
                <i class="fa fa-fw fa-paper-plane text-warning" v-if="t.specification.source.address === wallet.address"></i>
                <i class="fa fa-fw fa-inbox text-success" v-if="t.specification.source.address !== wallet.address"></i>
              </span> -->
              <span class="pull-right"><b>{{ (parseFloat(t.txJson.Amount) / 1000 / 1000) | xrp }} XRP</b></span>
            </li>
          </ul>
        </small>
        <br />
      </div>

      <div class="alert alert-warning text-center" v-if="offline">
        No transaction history available in offline mode
      </div>

      <h5 v-if="wallet.__tx.length > 0 && !offline"><b>Transactions</b></h5>

      <div class="well" v-if="!offline">
        <div class="alert alert-secondary text-center text-muted" v-if="!wallet.__txFetched">
          <i class="fa fa-spin fa-refresh"></i>
          <br />
          <small>Loading transactions...</small>
        </div>

        <p class="alert alert-warning text-center" v-if="wallet.__txFetched === 'actNotFound' && wallet.__tx.length < 1">
          Wallet not activated, deposit <b>{{ reserveXRP }}</b> XRP to activate this wallet.
        </p>
        <p class="alert alert-danger text-center" v-if="wallet.__txFetched !== false && wallet.__txFetched !== 'actNotFound' && wallet.__tx.length < 1">
          Error retrieving transactions<br /><b>{{ wallet.__txFetched !== true ? wallet.__txFetched : '' }}</b>
        </p>
        <small v-if="wallet.__tx.length > 0">
          <div class="list-group">
            <a @click="openTxDetails(t)" class="list-group-item list-group-item-action" v-for="t in wallet.__tx" v-bind:key="t.id" :class="{ 'list-group-item-warning' : activeTx == t.id }">
              <code class="text-primary">
                <b>{{ t.specification.source.address === wallet.address ? t.specification.destination.address : t.specification.source.address }}</b>
                <span v-if="t.specification.destination.tag" class="pull-right text-secondary"><i class="fa fa-tag"></i> {{ t.specification.destination.tag }}</span>
              </code>
              <br />
              {{ t.outcome.timestamp | localtime }}
              <span class="pull-right">
                &nbsp;
                <i class="fa fa-fw fa-paper-plane text-warning" v-if="t.specification.source.address === wallet.address"></i>
                <i class="fa fa-fw fa-inbox text-success" v-if="t.specification.source.address !== wallet.address"></i>
              </span>
              <span class="pull-right"><b>{{ t.outcome.deliveredAmount ? t.outcome.deliveredAmount.value : 0 | xrp }} {{ t.outcome.deliveredAmount ? t.outcome.deliveredAmount.currency : '' }}</b></span>
            </a>
          </div>
        </small>
      </div>

      <div class="text-center">
        <br />
        <br />
        <small><a @click="removeWallet()" class="text-danger"><i class="fa fa-times"></i> <u>Forget wallet</u></a></small>
      </div>

      <hr />

      <button @click="close()" class="btn btn-block btn-outline-primary"><i class="fa fa-arrow-left"></i> Back (overview)</button>
    </div>
  </div>
</template>

<script>
import swal from 'sweetalert'
import moment from 'moment-timezone'
import QrCode from '@xkeshi/vue-qrcode'
import TxDetails from './TxDetails'
import Modal from 'modal-vue'

export default {
  name: 'WalletDetails',
  props: [ 'close', 'ripple', 'wallet', 'exchangeRate', 'persistWallets' ],
  components: {
    QrCode,
    TxDetails,
    Modal
  },
  data () {
    return {
      activeTx: '',
      txDetails: null,
      walletSettings: false
    }
  },
  methods: {
    composeTx: function () {
      window.eventBus.composeTx({
        from: this.wallet.address
      })
    },
    removeWallet: function () {
      var $app = this
      swal({
        title: 'Are you sure?',
        text: 'This will remove the wallet from kyte. You can always add this wallet again.',
        icon: 'warning',
        buttons: true,
        dangerMode: true
      }).then((willDelete) => {
        if (willDelete) {
          $app.$parent.removeWallet($app.wallet)
          $app.close()
        }
      })
    },
    openTxDetails: function (tx) {
      this.txDetails = tx
      this.activeTx = tx.id
    },
    closeTxDetails: function () {
      this.txDetails = null
    }
  },
  computed: {
    lastLedgerVersion: function () {
      return window.eventBus.lastLedgerVersion
    },
    offline: function () {
      return window.eventBus.offline
    },
    tentativeTxs: function () {
      var $app = this
      return window.eventBus.txQueue.filter(function (r) {
        return r.from === $app.wallet.address
      })
    },
    reserveXRP: function () {
      if (!this.ripple || !this.ripple.__serverInfo || !this.ripple.__serverInfo.validatedLedger) return 0
      return this.ripple.__serverInfo.validatedLedger.reserveBaseXRP
    }
  },
  filters: {
    localtime: function (t) {
      var mnt = moment(t)
      return mnt.tz(moment.tz.guess()).format('YYYY-MM-DD HH:mm:ss')
    }
  },
  mounted: function () {
  }
}
</script>

<style lang="scss" scoped>
</style>
