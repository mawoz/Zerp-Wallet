<template>
  <div class="main">
    <h5>
      <b>Home</b>
      <span v-show="exchangeRate.bid" v-if="!offline" class="pull-right">
        <small><b class="text-muted"><span class="badge">1 XRP</span><span class="badge badge-primary">{{ exchangeRate.bid | currency }}</span></b></small>
      </span>
    </h5>

    <div class="list-group">
      <a @click="open(w)" class="list-group-item list-group-item-action" v-for="w in wallets" v-bind:key="w.address">
        <span class="pull-right" v-if="w.key"><i class="fa fa-paper-plane"></i></span>
        <code><b>{{ w.title }}</b></code>
        <br />
        <small v-if="w.title !== w.address"><code>{{ w.address }}</code></small>
        <h4 class="text-right" style="margin-bottom: 0; margin-top: 5px;" v-if="!offline">
          <span class="badge" :class="{ 'alert-secondary': w.balance === null, 'alert-warning': w.balance === 0, 'alert-success': w.balance > 0 }">
            <i v-if="w.balance === null" class="fa fa-spin fa-refresh"></i>
            <b v-if="w.balance !== null">{{ w.balance | xrp }}</b>
            XRP
          </span>
          <span class="text-muted pull-left" v-if="exchangeRate.bid && !offline">
            <small>
              <small>
                <b>{{ (w.balance * exchangeRate.bid) | currency }}</b>
              </small>
            </small>
          </span>
        </h4>
      </a>
    </div>

    <!-- eventBus: window.eventBus,
    closedLedgers: []
    {{ eventBus.lastLedger.ledgerVersion }}
    {{ closedLedgers }}
    var $app = this
    window.eventBus.$on('ledger', function (l) {
      $app.closedLedgers.push(l.ledgerVersion)
    })
    -->
  </div>
</template>

<script>

export default {
  name: 'WalletOverview',
  props: [ 'ripple', 'wallets', 'open', 'exchangeRate' ],
  components: {
  },
  data () {
    return {
    }
  },
  methods: {
  },
  computed: {
    offline: function () {
      return window.eventBus.offline
    }
  },
  mounted: function () {
  }
}
</script>

<style lang="scss" scoped>
</style>
