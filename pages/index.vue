<template>
  <v-tabs v-model="activeTab" @change="tabbed">
    <v-tab ripple v-for="(tab, index) in layout" :key="index">
      {{tab.tabName}}
    </v-tab>
    <v-tab-item v-for="(tab, index) in layout" :key="index" class="pt-2">
      <v-layout row wrap>
        <template v-for="(widget, index) in tab.widgets">
          <v-flex :key="index" v-bind:class="widget.width">
            <v-card class="mb-4 mx-2 pa-1 elevation-8 height-initial"
                   :style="rendered ? 'display: block' : 'display: none'">
              <div ref="views" style="overflow: hidden;"></div>
            </v-card>
          </v-flex>
        </template>
      </v-layout>
    </v-tab-item>
  </v-tabs>
</template>

<script>
import DataLoader from '~/api/backend.js'
const vega = require('vega')

export default {
  async asyncData ({ $axios, store }) {
    const loader = new DataLoader($axios)
    const layout = await loader.getLayout()
    return {
      layout, activeTab: 0, rendered: false
    }
  },
  methods: {
    async tabbed () {
      this.$store.commit('setControls', this.layout[this.activeTab].controls)

      this.rendered = false
      const widgets = this.layout[this.activeTab].widgets
      for (let index = 0; index < widgets.length; ++index) {
        const spec = widgets[index].spec
        let refIndex = index
        let iterator = 0

        // find view ref index
        while (iterator < this.activeTab) {
          refIndex += this.layout[iterator].widgets.length
          ++iterator
        }
        const wrap = this.$refs.views[refIndex].parentElement
        let view = await new vega.View({
          ...vega.parse(spec),
          height: undefined,
          autosize: {
            type: 'fit',
            contains: 'content',
            resize: true
          }
        }, {
          renderer: 'svg',
          container: this.$refs.views[refIndex],
          wrapperElem: wrap,
          responsive: true,
          widthHeightRatio: spec.width / spec.height,
          hover: true
        }).resize()

        const updater = async () => {
          const margin = 40
          if (wrap) {
            await view
              .width(wrap.offsetWidth - margin)
              .height((wrap.offsetWidth - margin) / spec.width * spec.height)
              .initialize(wrap)
              .runAsync('enter')
          }
        }

        setTimeout(updater, 300)
        window.addEventListener('resize', updater)
      }

      this.rendered = true
    }
  },
  async mounted () {
    await this.tabbed()
  }
}
</script>

<style>
.height-initial {
  min-height: 100px !important;
}
</style>
