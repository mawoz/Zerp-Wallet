<template>
  <div class="main">
    <div class="alert alert-primary text-center">
      <b>{{ tx.outcome.deliveredAmount ? tx.outcome.deliveredAmount.value : 0 | xrp }} {{ tx.outcome.deliveredAmount ? tx.outcome.deliveredAmount.currency : '' }}</b>
    </div>

    <ul class="list-group">
      <li class="list-group-item" :class="{ 'active' : tx.specification.source.address !== wallet.address }" @click="composeTx(tx.specification.source.address)">
        <label><small><b>From</b></small></label>
        <code class="tx" :class="{ 'text-white' : tx.specification.source.address !== wallet.address }">
          {{ tx.specification.source.address }}
          <span class="pull-right" v-if="tx.specification.source.address !== wallet.address"><i class="fa fa-paper-plane fa-fw"></i></span>
        </code>
      </li>
      <li class="list-group-item" :class="{ 'active' : tx.specification.destination.address !== wallet.address }" @click="composeTx(tx.specification.destination.address, tx.specification.destination.tag)">
        <label><small><b :class="{ 'text-warning' : tx.specification.destination.address !== wallet.address }">To</b></small></label>
        <code class="tx" :class="{ 'text-white' : tx.specification.destination.address !== wallet.address }">
          {{ tx.specification.destination.address }}
          <span class="pull-right" v-if="tx.specification.destination.address !== wallet.address"><i class="fa fa-paper-plane fa-fw"></i></span>
        </code>
        <small v-if="tx.specification.destination.tag">
          <code class="tx" :class="{ 'text-white' : tx.specification.destination.address !== wallet.address }"><i class="fa fa-tag"></i> {{ tx.specification.destination.tag }}</code>
        </small>
      </li>
      <li class="list-group-item">
        <label><small><b>Moment</b></small></label>
        <code class="tx">{{ tx.outcome.timestamp | localtime }}</code>
      </li>
      <li class="list-group-item">
        <label><small><b>Fee</b></small></label>
        <code class="tx">{{ tx.outcome.fee | xrp }} XRP</code>
      </li>
      <li class="list-group-item">
        <label><small><b>Ledger</b></small></label>
        <code class="tx">{{ tx.outcome.ledgerVersion }}</code>
      </li>
      <li class="list-group-item">
        <label><small><b>Transaction ID</b></small></label>
        <code class="tx">{{ tx.id }}</code>
      </li>
    </ul>

  </div>
</template>

<script>
import moment from 'moment-timezone'

export default {
  name: 'TxDetails',
  props: [ 'tx', 'wallet' ],
  components: {
  },
  data () {
    return {
    }
  },
  methods: {
    composeTx: function (address, tag) {
      // if (address !== this.wallet.address && this.wallet.tx) {
      window.eventBus.composeTx({
        from: this.wallet.address,
        to: address,
        tag: tag || ''
      })
      // }
    }
  },
  computed: {
  },
  mounted: function () {
  },
  filters: {
    localtime: function (t) {
      var mnt = moment(t)
      return mnt.tz(moment.tz.guess()).format('YYYY-MM-DD HH:mm:ss')
    }
  }
}
</script>

<style lang="scss" scoped>
  code {
    &.tx {
       overflow: hidden;
       text-overflow: ellipsis;
       white-space: nowrap;
       display: block;
    }
  }

  ul {
    li {
      padding: 4px 10px;

      &.list-group-item {
        label {
          margin-bottom: 0px;
        }
      }

      .compose-tx {
        padding: 0px 4px;
      }
    }
  }
</style>
