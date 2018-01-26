<template>
  <div class="passphrase">
    <span @click="switchViewmode()" class="view-mode" :class="{ 'text-muted': viewMode === 'password', 'text-primary': viewMode !== 'password' }"><i class="fa fa-eye fa-fw"></i></span>
    <span class="passphrase-wordcount" :class="{ 'bg-danger': passphraseWordCount < minWords, 'bg-warning': passphraseWordCount >= minWords && passphraseWordCount < safeWords, 'bg-success' : passphraseWordCount >= safeWords }">{{ passphraseWordCount }}</span>
    <input autocomplete="off" autocorrect="off" autocapitalize="off" spellcheck="false" :type="viewMode" class="form-control passphrase" :xclass="{ 'is-invalid': passphraseWordCount < minWords }" v-model="passphrase" :placeholder="placeholderTxt" autofocus :disabled="disabled" />
  </div>
</template>

<script>
export default {
  name: 'Passphrase',
  props: [ 'value', 'length', 'safelength', 'placeholder', 'disabled' ],
  data () {
    return {
      passphrase: this.value,
      viewMode: 'password'
    }
  },
  watch: {
    passphrase: function () {
      this.$emit('input', this.passphraseWordCount >= this.minWords ? this.formattedPassphrase : '')
    }
  },
  computed: {
    placeholderTxt: function () {
      if (typeof this.placeholder === 'string') {
        return this.placeholder
      }
      return 'Type your passphrase (min. ' + this.minWords + ' words)'
    },
    minWords: function () {
      return parseInt(this.length || 6)
    },
    safeWords: function () {
      return parseInt(this.safelength || 10)
    },
    formattedPassphrase: function () {
      return this.passphrase.toLowerCase().replace(/[^a-zA-Z\u00C0-\u017F ]/g, ' ').replace(/[ ]+/g, ' ').trim()
    },
    passphraseWordCount: function () {
      return this.formattedPassphrase.split(' ').filter((w) => {
        return w.length > 1
      }).length
    }
  },
  methods: {
    switchViewmode: function () {
      this.viewMode = this.viewMode === 'password' ? 'text' : 'password'
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss" scoped>
  i.fa {
    &.fa-eye {
      cursor: pointer;
    }
  }
  input.passphrase {
    padding-right: 62px;
  }
  span.view-mode {
    padding: 0px 10px;
    font-weight: bold;
    border-radius: 4px;
    position: absolute;
    margin-top: 8px;
    right: 22px;
    margin-right: 24px;
    color: white;
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
    cursor: default;
  }
</style>
