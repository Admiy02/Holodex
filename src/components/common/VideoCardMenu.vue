<template>
  <v-list v-if="video" dense>
    <v-list-item target="_blank" :href="`https://youtu.be/${video.id}`" @click.stop>
      <v-icon left>
        {{ icons.mdiYoutube }}
      </v-icon>
      {{ $t("views.settings.redirectModeLabel") }}
    </v-list-item>

    <v-list-item v-if="video.status === 'upcoming'" @click.prevent.stop="openGoogleCalendar">
      <v-icon left>
        {{ icons.mdiCalendar }}
      </v-icon>
      {{ $t("component.videoCard.googleCalendar") }}
    </v-list-item>
    <v-list-item :to="`/edit/video/${video.id}${video.type !== 'stream' ? '/mentions' : '/'}`">
      <v-icon left>
        {{ icons.mdiPencil }}
      </v-icon>
      {{ $t("component.videoCard.edit") }}
    </v-list-item>
    <template v-if="video.type !== 'clip'">
      <v-list-item :to="`/multiview/AAUY${video.id}%2CUAEYchat`">
        <v-icon left>
          {{ icons.mdiViewDashboard }}
        </v-icon>
        {{ $t("component.mainNav.multiview") }}
      </v-list-item>
    </template>

    <v-list-item :class="doneCopy ? 'green lighten-2' : ''" @click.stop="copyLink">
      <v-icon left>
        {{ icons.mdiClipboardPlusOutline }}
      </v-icon>
      {{ $t("component.videoCard.copyLink") }}
    </v-list-item>

    <v-list-item @click="$store.commit('setReportVideo', video)">
      <v-icon left>
        {{ icons.mdiFlag }}
      </v-icon>
      {{ $t("component.reportDialog.title") }}
    </v-list-item>

    <template v-if="$store.getters['isSuperuser']">
      <!-- <v-list-item> -->
      <v-lazy>
        <watch-quick-editor :video="video" />
      </v-lazy>
      <!-- </v-list-item> -->
    </template>
  </v-list>
</template>

<script>
import { dayjs } from "@/utils/time";
import copyToClipboard from "@/mixins/copyToClipboard";
import WatchQuickEditor from "@/components/watch/WatchQuickEditor.vue";

export default {
    components: { WatchQuickEditor },
    mixins: [copyToClipboard],
    props: {
        video: {
            type: Object,
            required: true,
        },
    },
    methods: {
        // Open google calendar to add the time specified in the element
        openGoogleCalendar() {
            const startdate = this.video.start_scheduled;
            const baseurl = "https://www.google.com/calendar/render?action=TEMPLATE&text=";
            const videoTitle = encodeURIComponent(this.video.title);
            const googleCalendarFormat = "YYYYMMDD[T]HHmmss";
            const eventStart = dayjs(startdate).format(googleCalendarFormat);
            const eventEnd = dayjs(startdate).add(1, "hour").format(googleCalendarFormat);
            const details = `<a href="${window.origin}/watch/${this.video.id}">Open Video</a>`;
            window.open(baseurl.concat(videoTitle, "&dates=", eventStart, "/", eventEnd, "&details=", details), "_blank");
        },
        copyLink() {
            const link = `${window.origin}/watch/${this.video.id}`;
            this.copyToClipboard(link);
        },
    },
};
</script>

<style></style>
