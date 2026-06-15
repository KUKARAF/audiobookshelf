<template>
  <div v-if="show" ref="wrapper" class="modal modal-bg w-full h-full fixed top-0 left-0 bg-primary/75 flex items-center justify-center z-60">
    <div ref="content" class="relative text-white w-96" v-click-outside="close">
      <div class="px-6 py-8 rounded-lg bg-bg shadow-lg border border-black-300">
        <h2 class="text-xl font-semibold mb-2">Send to eReader</h2>
        <p class="text-sm text-gray-400 mb-6">
          Open <a href="https://send.djazz.se" target="_blank" class="text-blue-400 underline">send.djazz.se</a> on your eReader's browser and enter the key shown there.
        </p>
        <ui-text-input-with-label v-model="code" label="Device Key" class="mb-6" @keyup.enter.native="submit" />
        <div class="flex items-center justify-end gap-3">
          <ui-btn color="bg-primary" :disabled="sending" @click="close">Cancel</ui-btn>
          <ui-btn color="bg-success" :disabled="sending || !code.trim()" @click="submit">
            {{ sending ? 'Sending...' : 'Send' }}
          </ui-btn>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      code: '',
      sending: false
    }
  },
  computed: {
    show: {
      get() {
        return this.$store.state.globals.showSendToEReaderModal
      },
      set(val) {
        this.$store.commit('globals/setShowSendToEReaderModal', val)
      }
    },
    item() {
      return this.$store.state.globals.sendToEReaderItem
    }
  },
  watch: {
    show(val) {
      if (val) {
        this.code = ''
        this.sending = false
        this.$nextTick(() => {
          const input = this.$el.querySelector('input')
          if (input) input.focus()
        })
      }
    }
  },
  methods: {
    close() {
      if (this.sending) return
      this.show = false
    },
    async submit() {
      const code = this.code.trim().toUpperCase()
      if (!code || !this.item) return

      this.sending = true
      try {
        const axios = this.$axios || this.$nuxt.$axios
        const response = await axios.get(`/api/items/${this.item.libraryItemId}/ebook`, {
          responseType: 'blob'
        })

        const blob = response.data
        const filename = `${this.item.title}.${this.item.ebookFormat}`
        const file = new File([blob], filename, { type: blob.type })

        const formData = new FormData()
        formData.append('key', code)
        formData.append('file', file, filename)
        formData.append('kepubify', 'on')

        await fetch('https://send.djazz.se/upload', {
          method: 'POST',
          body: formData,
          mode: 'no-cors'
        })

        this.$toast.success('Book sent to eReader!')
        this.show = false
      } catch (err) {
        console.error('Failed to send to eReader', err)
        this.$toast.error('Failed to send to eReader')
      } finally {
        this.sending = false
      }
    }
  }
}
</script>
