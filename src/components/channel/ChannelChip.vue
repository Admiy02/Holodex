<template>
  <!-- struggling a little bit with mobile/desktop compatibility -->
  <v-menu
    v-model="isHover"
    open-on-click
    open-on-focus
    :open-on-hover="!$store.state.isMobile"
    class=""
    top
    :nudge-top="size + 5"
    content-class="elevation-0"
    close-delay="250"
  >
    <template #activator="{ on: tooltip }">
      <v-avatar
        left
        :size="size"
        class="mr-1"
        v-on="tooltip"
      >
        <v-img
          :src="photo"
          crossorigin="anonymous"
          :alt="`${channel.name}'s profile picture`"
          :width="size"
          :height="size"
        />
        <!-- <span class="channel-name-overlay">{{ channelName }}</span> -->
        <slot :isHover="isHover">
          <v-fade-transition>
            <v-overlay v-show="isHover" absolute class="chip-overlay">
              <v-btn icon :to="`/channel/${channel.id}`">
                <v-icon>{{ icons.mdiLoginVariant }}</v-icon>
              </v-btn>
            </v-overlay>
          </v-fade-transition>
        </slot>
      </v-avatar>
    </template>
    <div class="channel-hover-tooltip">
      <ChannelSocials
        :channel="channel"
        vertical
        hide-yt
        hide-twitter
      />
      <span class="grey--text text--lighten-1 ml-2">{{ channelName }}</span>
    </div>
  </v-menu>
</template>

<script lang="ts">
import { resizeChannelPhoto } from "@/utils/functions";
import ChannelSocials from "@/components/channel/ChannelSocials.vue";

export default {
    name: "ChannelChip",
    components: {
        ChannelSocials,
    },
    props: {
        channel: {
            type: Object,
            required: true,
        },
        size: {
            type: Number,
            required: false,
            default: 60,
        },
        // close: {
        //     type: Boolean,
        //     required: false,
        // },
    },
    data() {
        return { isHover: false };
    },
    computed: {
        channelName() {
            const prop = this.$store.state.settings.nameProperty;
            if (this.channel[prop]) return this.channel[prop];
            return this.channel.name;
        },
        photo() {
            if (!this.channel.photo) return "";
            return resizeChannelPhoto(this.channel.photo, this.size);
        },
    },
};
</script>

<style>
.channel-hover-tooltip {
    background-color: rgba(58, 58, 58, 0.7);
    display: flex;
    flex-direction: row;
    align-items: baseline;
    padding: 0 8px 0 0;
    margin-top: -3px;
    max-height: 32px;
    overflow: hidden;
}
</style>
