<template>
  <model-card type="equipment" :context="context" :element="element" header-height="150px">
    <template #glance>
      <div v-if="context && context.component.slots && context.component.slots.glance" class="display-flex flex-direction-column align-items-flex-start">
        <generic-widget-component :context="childContext(slotComponent)" v-for="(slotComponent, idx) in context.component.slots.glance" :key="'glance-' + idx" @command="onCommand" />
      </div>
      <!-- <div class="equipment-stats" v-else><small v-if="element.equipment">{{element.equipment.length}}</small></div> -->
    </template>
    <div class="card-content-padding">
      <generic-widget-component :context="listContext" />
      <p>
        <f7-button fill round large card-close :color="color" class="margin-horizontal" :text="$t('home.cards.close')" />
      </p>
    </div>
  </model-card>
</template>

<style lang="stylus">
.equipment-card
  height 150px
.equipment-stats
  font-weight normal
</style>

<script>
import mixin from '@/components/widgets/widget-mixin'
import { equipmentListComponent } from '@/components/widgets/standard/list/default-list-item'
import CardMixin from './card-mixin'
import ModelCard from './model-card.vue'

export default {
  mixins: [mixin, CardMixin],
  props: ['tabContext'],
  components: {
    ModelCard
  },
  computed: {
    listContext () {
      return {
        store: this.$store.getters.trackedItems,
        component: equipmentListComponent(this.element.equipment, this.tabContext, false)
      }
    }
  }
}
</script>
