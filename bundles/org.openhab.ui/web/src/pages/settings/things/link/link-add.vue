<template>
  <f7-page @page:afterin="onPageAfterIn">
    <f7-navbar title="Link Channel to Item" back-link="Cancel">
      <f7-nav-right class="if-not-aurora">
        <f7-link @click="save()" v-if="$theme.md" icon-md="material:save" icon-only />
        <f7-link @click="save()" v-if="!$theme.md">
          Link
        </f7-link>
      </f7-nav-right>
    </f7-navbar>
    <f7-block class="block-narrow">
      <f7-col v-if="channel">
        <f7-block-title>Channel</f7-block-title>
        <f7-list media-list>
          <f7-list-item media-item class="channel-item"
                        :title="channel.label || channelType.label"
                        :footer="channel.description || channelType.description"
                        :subtitle="channel.uid + ' (' + getItemType(channel) + ')'" />
        </f7-list>
      </f7-col>

      <div v-if="!item && !items" class="text-align-center">
        <f7-preloader />
        <div>Loading...</div>
      </div>
      <template v-if="!item && items">
        <!-- Option to create new item (if not supplied by prop) -->
        <f7-col>
          <f7-block-title>Item</f7-block-title>
          <f7-list media-list>
            <f7-list-item radio :checked="!createItem" value="false" @change="createItem = false" title="Use an existing Item" name="item-creation-choice" />
            <f7-list-item radio :checked="createItem" value="true" @change="createItem = true" title="Create a new Item" name="item-creation-choice" />
          </f7-list>
        </f7-col>

        <!-- Choose item to link -->
        <f7-col v-if="!createItem">
          <f7-list>
            <item-picker key="itemLink" title="Item to Link" name="item" :value="selectedItemName" :multiple="false" :items="items" :filterType="getCompatibleItemTypes()"
                         @input="(value) => selectedItemName = value" />
          </f7-list>
        </f7-col>

        <!-- Create new item -->
        <f7-col v-else>
          <item-form :item="newItem" :items="items" :enable-name="true" @valid="itemValid = $event" />
          <f7-list>
            <item-picker key="newItem-groups" title="Parent Group(s)" name="parent-groups" :value="newItem.groupNames" :items="items" @input="(value) => newItem.groupNames = value" :multiple="true" filterType="Group" />
          </f7-list>
        </f7-col>
      </template>

      <!-- Item to link supplied as prop -->
      <f7-col v-else-if="item">
        <f7-block-title>Item</f7-block-title>
        <f7-list media-list>
          <ul>
            <item :item="item" />
          </ul>
        </f7-list>
        <f7-block-title>Thing</f7-block-title>
        <f7-list inline-labels no-hairlines-md>
          <thing-picker title="Thing" name="thing" :value="selectedThingId" @input="(e) => selectedThingId = e" />
        </f7-list>
        <div v-if="selectedThing.UID && selectedThingType.UID">
          <f7-block-title>Channel</f7-block-title>
          <channel-list :thing="selectedThing" :thingType="selectedThingType"
                        :picker-mode="true" :item-type-filter="item.type" :channel-types="selectedThingChannelTypes"
                        @selected="(channel) => loadProfileTypes(channel)" />
        </div>
      </f7-col>

      <f7-block v-if="!ready && !(!item && !items)" class="text-align-center">
        <f7-preloader />
        <div>Loading...</div>
      </f7-block>

      <!-- Profile configuration -->
      <f7-col v-else-if="profileTypes.length && currentItem">
        <f7-block-title>Profile</f7-block-title>
        <f7-block-footer class="padding-left padding-right">
          Profiles define how Channels and Items work together. Install transformation add-ons to get additional profiles.
          <f7-link external color="blue" target="_blank" href="https://www.openhab.org/link/profiles">
            Learn more about profiles.
          </f7-link>
        </f7-block-footer>
        <f7-list class="profile-list profile-disabled">
          <f7-list-item radio v-for="profileType in profileTypes" class="profile-item"
                        :checked="!currentProfileType && profileType.uid === 'system:default' || currentProfileType && profileType.uid === currentProfileType.uid"
                        :disabled="!compatibleProfileTypes.includes(profileType)"
                        :class="{ 'profile-disabled': !compatibleProfileTypes.includes(profileType) }"
                        @change="onProfileTypeChange(profileType.uid)"
                        :key="profileType.uid" :title="profileType.label" name="profile-type" />
        </f7-list>
      </f7-col>
      <f7-col v-if="profileTypeConfiguration != null">
        <f7-block-title>Profile Configuration</f7-block-title>
        <config-sheet ref="profileConfiguration"
                      :key="'profileTypeConfiguration-' + currentProfileType.uid"
                      :parameter-groups="profileTypeConfiguration.parameterGroups"
                      :parameters="profileTypeConfiguration.parameters"
                      :configuration="configuration" />
      </f7-col>
    </f7-block>

    <div v-if="ready && profileTypes.length" class="if-aurora display-flex justify-content-center padding margin">
      <div class="flex-shrink-0">
        <f7-button class="padding-left padding-right" style="width: 150px" color="blue" large raised fill @click="save">
          Link
        </f7-button>
      </div>
    </div>
  </f7-page>
</template>

<style lang="stylus">
.profile-list
  .profile-item.profile-disabled
    pointer-events none
    .icon-radio
      opacity 0.3
    .item-title
      opacity 0.55
</style>

<script>
import diacritic from 'diacritic'

import ConfigSheet from '@/components/config/config-sheet.vue'
import ItemPicker from '@/components/config/controls/item-picker.vue'
import ThingPicker from '@/components/config/controls/thing-picker.vue'
import ChannelList from '@/components/thing/channel-list.vue'
import ItemForm from '@/components/item/item-form.vue'

import Item from '@/components/item/item.vue'

import * as Types from '@/assets/item-types.js'
import * as SemanticClasses from '@/assets/semantics.js'

export default {
  components: {
    ConfigSheet,
    ItemPicker,
    ThingPicker,
    Item,
    ChannelList,
    ItemForm
  },
  props: ['thing', 'channel', 'channelType', 'item'],
  data () {
    return {
      ready: true,
      createItem: false,
      items: null,
      itemValid: true,
      link: {
        itemName: null,
        channelUID: null,
        configuration: {}
      },
      selectedItemName: null,
      selectedThingId: '',
      selectedThing: {},
      selectedThingType: {},
      selectedThingChannelTypes: {},
      selectedChannel: null,
      profileTypes: [],
      currentProfileType: null,
      profileTypeConfiguration: null,
      newItem: {},
      configuration: {},
      types: Types,
      semanticClasses: SemanticClasses
    }
  },
  created () {
    if (!this.item) {
      this.$oh.api.get('/rest/items').then((items) => {
        this.items = items
      })
    }
  },
  computed: {
    currentItem () {
      return this.item ? this.item : this.createItem ? this.newItem : this.items ? this.items.find(item => item.name === this.selectedItemName) : null
    },
    compatibleProfileTypes () {
      let currentItemType = this.currentItem && this.currentItem.type ? this.currentItem.type : ''
      return this.profileTypes.filter(p => !p.supportedItemTypes.length || p.supportedItemTypes.includes(currentItemType.split(':', 1)[0]))
    }
  },
  methods: {
    onPageAfterIn (event) {
      if (!this.channel) return
      this.loadProfileTypes(this.channel)
      let newItemName = diacritic.clean(this.thing.label).replace(/[^0-9a-z]/gi, '')
      newItemName += '_'
      newItemName += (this.channel.label) ? diacritic.clean(this.channel.label).replace(/[^0-9a-z]/gi, '') : diacritic.clean(this.channelType.label).replace(/[^0-9a-z]/gi, '')
      const defaultTags = (this.channel.defaultTags.length > 0) ? this.channel.defaultTags : this.channelType.tags
      this.$set(this, 'newItem', {
        name: newItemName,
        label: this.channel.label || this.channelType.label,
        category: (this.channelType) ? this.channelType.category : '',
        groupNames: [],
        type: this.channel.itemType || 'Switch',
        tags: (defaultTags.find((t) => SemanticClasses.Points.indexOf(t) >= 0)) ? defaultTags : [...defaultTags, 'Point']
      })
    },
    loadProfileTypes (channel) {
      this.ready = false
      this.selectedChannel = channel
      const getProfileTypes = this.$oh.api.get('/rest/profile-types?channelTypeUID=' + channel.channelTypeUID)
      getProfileTypes.then((data) => {
        this.profileTypes = data
        this.profileTypes.unshift(data.splice(data.findIndex(p => p.uid === 'system:default'), 1)[0]) // move default to be first
        this.ready = true
      })
    },
    onProfileTypeChange (profileTypeUid) {
      if (!profileTypeUid) {
        this.profileTypeConfiguration = null
        this.currentProfileType = null
        return
      }
      this.currentProfileType = this.profileTypes.find((p) => p.uid === profileTypeUid)
      const getProfileConfigDescription = this.$oh.api.get('/rest/config-descriptions/profile:' + profileTypeUid)
      getProfileConfigDescription.then((data) => {
        this.profileTypeConfiguration = data
      }).catch((err) => {
        // just clear out the config sheet
        console.warn(`No configuration for profile type ${profileTypeUid}: ` + err)
        this.profileTypeConfiguration = null
      })
    },
    getItemType (channel) {
      if (channel && channel.kind === 'TRIGGER') return 'Trigger'
      if (!channel || !channel.itemType) return '?'
      return channel.itemType
    },
    getCompatibleItemTypes () {
      let compatibleItemTypes = []
      if (this.channel.itemType) {
        compatibleItemTypes.push(this.channel.itemType)
        if (this.channel.itemType.indexOf('Number:') === 0) { compatibleItemTypes.push('Number') }
        if (this.channel.itemType === 'Color') { compatibleItemTypes.push('Switch', 'Dimmer') }
        if (this.channel.itemType === 'Dimmer') { compatibleItemTypes.push('Switch') }
      }
      return compatibleItemTypes
    },
    save () {
      const link = {}
      if (this.channel) {
        link.channelUID = this.channel.uid
      } else if (this.selectedChannel) {
        link.channelUID = this.selectedChannel.uid
      }

      link.itemName = this.currentItem.name

      link.configuration = Object.assign({}, this.configuration)
      if (this.currentProfileType) {
        link.configuration.profile = this.currentProfileType.uid
      }

      // checks
      if (this.createItem && !this.itemValid) {
        this.$f7.dialog.alert('Please correct the item to link')
        return
      }
      if (!link.itemName) {
        this.$f7.dialog.alert('Please configure the item to link')
        return
      }
      if (!link.channelUID) {
        this.$f7.dialog.alert('Please configure the channel to link')
        return
      }
      if (this.$refs.profileConfiguration && !this.$refs.profileConfiguration.isValid()) {
        this.$f7.dialog.alert('Please review the profile configuration and correct validation errors')
        return
      }

      if ((this.channel ? this.channel : this.selectedChannel).kind === 'TRIGGER') {
        if (!this.compatibleProfileTypes.length) {
          this.$f7.dialog.alert('There is no profile available for the selected item')
          return
        }
        if (!this.currentProfileType || !this.compatibleProfileTypes.includes(this.currentProfileType)) {
          this.$f7.dialog.alert('Please configure a valid profile')
          return
        }
      }

      if (this.createItem) {
        this.$oh.api.put('/rest/items/' + this.newItem.name, this.newItem).then((data) => {
          this.$oh.api.put('/rest/links/' + link.itemName + '/' + encodeURIComponent(link.channelUID), link).then((data) => {
            this.$f7.toast.create({
              text: 'Item and link created',
              destroyOnClose: true,
              closeTimeout: 2000
            }).open()
            this.$f7router.back()
          })
        })
      } else {
        this.$oh.api.put('/rest/links/' + link.itemName + '/' + encodeURIComponent(link.channelUID), link).then((data) => {
          this.$f7.toast.create({
            text: 'Link created',
            destroyOnClose: true,
            closeTimeout: 2000
          }).open()
          this.$f7router.back()
        })
      }
    }
  },
  watch: {
    selectedThingId () {
      this.selectedThing = {}
      this.selectedThingType = {}
      this.profileTypes = []
      this.currentProfileType = null
      this.profileTypeConfiguration = null
      this.ready = false
      if (!this.selectedThingId) return
      this.$oh.api.get('/rest/things/' + this.selectedThingId).then((data) => {
        this.selectedThing = data

        let typePromises = [this.$oh.api.get('/rest/thing-types/' + this.selectedThing.thingTypeUID),
          this.$oh.api.get('/rest/channel-types?prefixes=system,' + this.selectedThing.thingTypeUID.split(':')[0])]

        Promise.all(typePromises).then(data2 => {
          this.selectedThingType = data2[0]
          this.selectedThingChannelTypes = data2[1]
          this.ready = true
        })
      })
    },
    currentItem () {
      if (this.currentProfileType && !this.compatibleProfileTypes.find(p => p.uid === this.currentProfileType.uid)) {
        this.currentProfileType = null
      }
    }
  }
}
</script>
