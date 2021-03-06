<template lang="pug">
.container#Vocab
  .field
    .control
      input.input(name="q" placeholder="Type here to search." v-model="q" @keydown.enter="onQChange()")
  .columns
    .column.is-6.entry-display
      .font-hanzi(style="font-size: 120px; min-height: 200px; position: relative;")
        .clickable.text-center(
          @contextmenu.prevent="(evt) => { selectedVocab = simplified; $refs.vocabContextmenu.open(evt) }"
        ) {{simplified}}
        b-loading(:active="isQLoading" :is-full-page="false")
      .buttons.has-addons
        button.button(@click="i--" :disabled="i < 1") Previous
        button.button(@click="i++" :disabled="i > entries.length - 2") Next
        b-dropdown(hoverable aria-role="list")
          button.button(slot="trigger")
            fontawesome(icon="caret-down")
          b-dropdown-item(aria-role="listitem") Search in MDBG
    .column.is-6
      b-collapse.card(animation="slide" style="margin-bottom: 1em;" :open="typeof current === 'object'")
        .card-header(slot="trigger" slot-scope="props" role="button")
          h2.card-header-title Reading
          a.card-header-icon
            fontawesome(:icon="props.open ? 'caret-down' : 'caret-up'")
        .card-content
          span {{current.pinyin}}
      b-collapse.card(animation="slide" style="margin-bottom: 1em;" :open="!!current.traditional")
        .card-header(slot="trigger" slot-scope="props" role="button")
          h2.card-header-title Traditional
          a.card-header-icon
            fontawesome(:icon="props.open ? 'caret-down' : 'caret-up'")
        .card-content
          .font-hanzi.clickable(style="font-size: 60px; height: 80px;"
            @contextmenu.prevent="(evt) => { selectedVocab = current.traditional; $refs.vocabContextmenu.open(evt) }"
          ) {{current.traditional}}
      b-collapse.card(animation="slide" style="margin-bottom: 1em;" :open="typeof current === 'object'")
        .card-header(slot="trigger" slot-scope="props" role="button")
          h2.card-header-title English
          a.card-header-icon
            fontawesome(:icon="props.open ? 'caret-down' : 'caret-up'")
        .card-content
          span {{current.english}}
      b-collapse.card(animation="slide" style="margin-bottom: 1em;" :open="sentences.length > 0")
        .card-header(slot="trigger" slot-scope="props" role="button")
          h2.card-header-title Sentences
          a.card-header-icon
            fontawesome(:icon="props.open ? 'caret-down' : 'caret-up'")
        .card-content
          div(v-for="s, i in sentences" :key="i")
            span.clickable.sentence-part(
              @contextmenu.prevent="(evt) => { selectedSentence = s.chinese; $refs.sentenceContextmenu.open(evt) }"
            ) {{s.chinese}}&nbsp;
            span.sentence-part {{s.english}}
  vue-context(ref="vocabContextmenu" lazy)
    li
      a(role="button" @click.prevent="speak(selectedVocab)") Speak
    li(v-if="!vocabIds[selectedVocab] || !vocabIds[selectedVocab].length")
      a(role="button" @click.prevent="addToQuiz(selectedVocab, 'vocab')") Add to quiz
    li(v-else)
      a(role="button" @click.prevent="removeFromQuiz(selectedVocab, 'vocab')") Remove from quiz
    li
      router-link(:to="{ path: '/vocab', query: { q: selectedVocab } }" target="_blank") Search for vocab
    li
      router-link(:to="{ path: '/hanzi', query: { q: selectedVocab } }" target="_blank") Search for Hanzi
    li
      a(:href="`https://www.mdbg.net/chinese/dictionary?page=worddict&wdrst=0&wdqb=*${selectedVocab}*`"
        target="_blank" rel="noopener") Open in MDBG
  vue-context(ref="sentenceContextmenu" lazy)
    li
      a(role="button" @click.prevent="speak(selectedSentence)") Speak
    li(v-if="!sentenceIds[selectedSentence] || !sentenceIds[selectedSentence].length")
      a(role="button" @click.prevent="addToQuiz(selectedSentence, 'sentence')") Add to quiz
    li(v-else)
      a(role="button" @click.prevent="removeFromQuiz(selectedSentence, 'sentence')") Remove from quiz
    li
      router-link(:to="{ path: '/vocab', query: { q: selectedSentence } }" target="_blank") Search for vocab
    li
      router-link(:to="{ path: '/hanzi', query: { q: selectedSentence } }" target="_blank") Search for Hanzi
    li
      a(:href="`https://www.mdbg.net/chinese/dictionary?page=worddict&wdrst=0&wdqb=${selectedSentence}`"
        target="_blank" rel="noopener") Open in MDBG
</template>

<script lang="ts">
import { Vue, Component, Watch } from 'vue-property-decorator'
import XRegExp from 'xregexp'
import { AxiosInstance } from 'axios'

import { speak } from '../utils'

@Component
export default class Vocab extends Vue {
  entries: string[] = []
  i: number = 0
  isQLoading = false

  sentences: any[] = []

  selectedVocab = ''
  selectedSentence = ''

  vocabIds: any = {}
  sentenceIds: any = {}

  q = ''

  speak = speak

  get current () {
    return this.entries[this.i] || '' as any
  }

  get simplified () {
    return typeof this.current === 'string' ? this.current : this.current.simplified
  }

  created () {
    const q = this.$route.query.q
    this.q = (Array.isArray(q) ? q[0] : q) || ''
    this.onQChange()
  }

  async getApi (silent = true) {
    return await this.$store.dispatch('getApi', silent) as AxiosInstance
  }

  @Watch('$store.state.user')
  async onQChange () {
    if (this.$route.query.q !== this.q) {
      this.$router.push({ query: { q: this.q } })
    }

    if (this.q && this.$store.state.user) {
      this.isQLoading = true
      const api = await this.getApi()

      let qs = (await api.post('/api/lib/jieba', { entry: this.q })).data.result as string[]
      qs = qs.filter(h => XRegExp('\\p{Han}+').test(h))
      this.$set(this, 'entries', qs.filter((h, i) => qs.indexOf(h) === i))
      this.loadContent()
    }

    this.i = 0
    this.isQLoading = false
  }

  loadContent () {
    this.loadVocab()
    this.loadSentences()
  }

  @Watch('current')
  async loadVocab () {
    if (typeof this.current === 'string') {
      const api = await this.getApi()

      const vs = (await api.post('/api/vocab/match', { entry: this.current })).data.result as any[]

      if (vs.length > 0) {
        this.entries = [
          ...this.entries.slice(0, this.i),
          ...vs,
          ...this.entries.slice(this.i + 1)
        ]
      }
    }
  }

  @Watch('simplified')
  async loadSentences () {
    const api = await this.getApi()
    const ss = (await api.post('/api/sentence/q', { entry: this.simplified })).data.result as any[]
    this.$set(this, 'sentences', ss)
  }

  @Watch('selectedVocab')
  async loadVocabStatus () {
    if (this.selectedVocab) {
      const api = await this.getApi()
      const r = await api.post('/api/card/q', {
        cond: { item: this.selectedVocab, type: 'vocab' },
        hasCount: false,
        projection: {
          _id: 1
        }
      })
      this.$set(this.vocabIds, this.selectedVocab, r.data.result.map((d: any) => d._id))
    }
  }

  @Watch('selectedSentence')
  async loadSentenceStatus () {
    if (this.selectedSentence) {
      const api = await this.getApi()
      const r = await api.post('/api/card/q', {
        cond: { item: this.selectedSentence, type: 'sentence' },
        hasCount: false,
        projection: {
          _id: 1
        }
      })
      this.$set(this.sentenceIds, this.selectedSentence, r.data.result.map((d: any) => d._id))
    }
  }

  async addToQuiz (item: string, type: string) {
    const api = await this.getApi()
    await api.put('/api/card/', { item, type })
    this.$buefy.snackbar.open(`Added ${type}: ${item} to quiz`)

    type === 'vocab' ? this.loadVocabStatus() : this.loadSentenceStatus()
  }

  async removeFromQuiz (item: string, type: string) {
    const api = await this.getApi()
    const ids = (type === 'vocab' ? this.vocabIds[item] : this.sentenceIds[item]) || []
    await Promise.all(ids.map((id: string) => api.delete('/api/card/', {
      data: { id }
    })))

    this.$buefy.snackbar.open(`Removed ${type}: ${item} from quiz`)

    type === 'vocab' ? this.loadVocabStatus() : this.loadSentenceStatus()
  }
}
</script>

<style lang="scss">
#Vocab {
  .entry-display {
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .card-content {
    max-height: calc(100vh - 500px);
    overflow: scroll;
  }
}
</style>
