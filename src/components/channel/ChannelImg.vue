<template>
  <!-- Render with opaque response for cache if size is lte 40 -->
  <v-lazy
    v-if="noLink"
    tag="img"
    :src="photo"
    crossorigin="anonymous"
    loading="lazy"
    :alt="`${channel.name}'s profile picture`"
    :width="size"
    :height="size"
    class="d-block"
  />
  <a v-else :href="`/channel/${channel.id}`" @click.exact.stop.prevent="goToChannel()">
    <v-lazy
      tag="img"
      :src="photo"
      crossorigin="anonymous"
      loading="lazy"
      :alt="`${channel.name}'s profile picture`"
      :width="size"
      :height="size"
      class="d-block"
      :class="rounded && 'rounded-circle'"
    />
  </a>
</template>

<script lang="ts">
import { resizeChannelPhoto } from "@/utils/functions";

export default {
    name: "ChannelImg",
    props: {
        channel: {
            type: Object,
            required: true,
        },
        size: {
            type: [String, Number],
            default: 40,
        },
        noLink: {
            type: Boolean,
            default: false,
        },
        rounded: {
            type: Boolean,
            default: false,
        },
    },
    computed: {
        photo() {
            if (!this.channel.photo) return "";
            return resizeChannelPhoto(this.channel.photo, this.size);
        },
    },
    methods: {
        goToChannel() {
            this.$router.push({ path: `/channel/${this.channel.id}` });
        },
    },
};
</script>

<style scoped>
img:hover {
    cursor: pointer;
}
</style>
