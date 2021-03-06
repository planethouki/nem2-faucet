<template lang="pug">
div
  section.section(v-if="drained")
    .container
      .notification.is-warning This faucet has been drained. Sorry for inconvenience.

  main.section
    form(@submit.prevent="claim")
      .container
        .columns
          .column.is-8
            b-field(label="Recipient")
              b-input(v-model="form.recipient"
                required
                maxlength="46"
                :pattern="recipientPattern"
                :title="recipientPlaceholder"
                :placeholder="recipientPlaceholder"
                :disabled="drained"
              )
          .column.is-4
            b-field(label="Amount")
              b-input(v-model.number="form.amount" type="number"
                min="0"
                :max="outOpt"
                :step="step"
                :placeholder="amountPlaceholder"
                :disabled="drained"
              )

        .columns
          .column.is-8
            b-field(label="Message")
              b-input(v-model="form.message"
                maxlength="1023"
                placeholder="(Optional)"
                :disabled="drained"
              )
          .column.is-4
            .columns.is-mobile
              .column.is-6
                b-field(label="Encryption")
                  b-switch(v-model="form.encryption" style="margin-top:5px")
                  | {{ form.encryption ? 'Encrypted' : 'Plain' }}
              .column.is-6
                b-field(label="Submit")
                  button(type="submit" class="button is-primary is-fullwidth" :disabled="drained || waiting")
                    span CLAIM!

        .columns
          .column.is-8
            b-field(label="Faucet Address")
              b-input(:value="address" readonly :disabled="drained")
          .column.is-4
            b-field(label="Faucet Balance")
              b-input(:value="balance" type="number" readonly :disabled="drained")

  Readme(
    :publicUrl="publicUrl"
    :network="network"
    :mosaicId="mosaicId"
    :outMin="outMin"
    :outMax="outMax"
  )
</template>

<script>
import { mapActions, mapGetters } from 'vuex'
import { Address, Listener } from 'nem2-sdk'

import Readme from '~/components/Readme'

export default {
  name: 'Home',
  components: {
    Readme
  },
  data() {
    return {
      form: {
        recipient: null,
        message: null,
        amount: null,
        encryption: false
      },
      drained: true,
      waiting: false,
      network: null,
      apiUrl: null,
      publicUrl: null,
      mosaicId: null,
      outMin: null,
      outMax: null,
      outOpt: null,
      step: null,
      address: null,
      balance: null,
      recipientPattern: null,
      recipientPlaceholder: null,
      amountPlaceholder: null
    }
  },
  computed: {
    ...mapGetters(['attributes', 'transactions'])
  },
  asyncData({ res, store, error }) {
    let attrs = store.getters.attributes
    if (res && res.data) {
      if (res.data.error) {
        return error(res.data.error)
      }
      attrs = { ...attrs, ...res.data.attributes }
      store.dispatch('setAttributes', { attributes: attrs })
    }
    const firstChar = attrs.address[0]
    const recipientPattern = `^${firstChar}[ABCD].+`
    const recipientPlaceholder = `${attrs.network} address start with a capital ${firstChar}`
    const amountPlaceholder = `(Up to ${attrs.outOpt}. Optional, if you want fixed amount)`
    const data = {
      ...attrs,
      recipientPattern,
      recipientPlaceholder,
      amountPlaceholder
    }
    console.debug('asyncData: %o', data)
    return data
  },
  async mounted() {
    const faucetAddress = Address.createFromRawAddress(this.address)
    this.listener = new Listener(this.publicUrl.replace('http', 'ws'), WebSocket)
    this.listener.open().then(() => {
      // prettier-ignore
      this.listener.unconfirmedAdded(faucetAddress)
        .subscribe(_ => {
          this.info('Your request had been unconfirmed status!')
        })
      // prettier-ignore
      this.listener.confirmed(faucetAddress)
        .subscribe(_ => {
          this.info('Your Request had been confirmed status!')
        })
    })

    if (this.$recaptcha) {
      await this.$recaptcha.init()
    }
    if (process.browser) {
      const { recipient, amount, message, encryption } = this.$nuxt.$route.query
      this.form = { ...this.form, recipient, amount, message, encryption }
    }
  },
  beforeDestroy() {
    this.listener.close()
  },
  methods: {
    ...mapActions(['setConfig', 'addTransaction']),
    async claim() {
      this.waiting = true
      this.$router.push({ path: this.$route.path, query: this.form })
      const formData = { ...this.form }
      if (this.$recaptcha) {
        formData.reCaptcha = await this.$recaptcha.execute('login')
      }
      this.$axios
        .$post('/claims', formData)
        .then(resp => {
          this.info(`Send your declaration.`)
          this.success(`Amount: ${resp.amount} ${this.mosaicId}`)
          this.success(`Transaction Hash: ${resp.txHash}`)
          const transaction = {
            hash: resp.txHash,
            mosaicId: this.mosaicId,
            amount: resp.amount,
            recipient: formData.recipient
          }
          this.addTransaction({ transaction })
        })
        .catch(err => {
          const msg =
            (err.response.data && err.response.data.error) ||
            err.response.statusTest
          this.failed(`Message from server: ${msg}`)
        })
        .finally(() => {
          this.waiting = false
        })
    },
    info(message) {
      this.$buefy.snackbar.open({
        type: 'is-info',
        message,
        queue: false
      })
    },
    success(message) {
      this.$buefy.snackbar.open({
        type: 'is-success',
        message,
        queue: false,
        duration: 8000
      })
    },
    warning(message) {
      this.$buefy.snackbar.open({
        type: 'is-warning',
        message,
        queue: false,
        duration: 8000
      })
    },
    failed(message) {
      this.$buefy.snackbar.open({
        type: 'is-danger',
        message,
        queue: false,
        duration: 8000
      })
    }
  }
}
</script>
