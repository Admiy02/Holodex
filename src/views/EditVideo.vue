<template>
  <v-container v-if="!isLoading && !hasError" fluid class="video-editor">
    <v-row>
      <v-col
        :md="3"
        :lg="4"
        cols="12"
        class="px-0 pt-0 px-md-3"
      >
        <WatchFrame :video="video">
          <template #youtube>
            <youtube
              v-if="video.id"
              :video-id="video.id"
              :player-vars="{
                ...(timeOffset && { start: timeOffset }),
                autoplay: 1,
                playsinline: 1,
              }"
              @ready="ready"
            />
          </template>
        </WatchFrame>
        <div v-if="video.comments.length" class="comment-scroller">
          <WatchComments
            v-if="video && video.comments && video.comments.length"
            key="comments"
            hide-buckets
            :comments="video.comments"
            :video="video"
            :limit="$store.state.isMobile ? 5 : 0"
            @timeJump="seekTo"
          />
        </div>
      </v-col>
      <v-col class="related-videos pt-0" :md="9" :lg="8">
        <v-alert
          v-if="!$store.state.userdata || !$store.state.userdata.jwt"
          color="error"
          v-html="$t('views.editor.needlogin')"
        />
        <v-row fluid>
          <v-tabs v-model="currentTab">
            <v-tab :disabled="video.type !== 'stream'">
              {{ $t("component.search.type.topic") }}
            </v-tab>
            <v-tab :disabled="video.type !== 'stream'">
              {{ $t("component.mainNav.music") }}
            </v-tab>
            <v-tab>{{ $t("views.editor.channelMentions.title") }}</v-tab>
            <v-tab disabled>
              {{ $t("views.editor.sources.title") }}
            </v-tab>
          </v-tabs>
          <v-col cols="12" class="pa-4">
            <div v-show="currentTab === TABS.TOPIC">
              <v-card-title>
                <v-icon left>
                  {{ icons.mdiAnimationPlay }}
                </v-icon>
                <h5>{{ $t("views.editor.changeTopic.title") }}</h5>
              </v-card-title>
              <v-card-text>
                <p>
                  {{ $t("views.editor.changeTopic.info") }}
                </p>
                <v-autocomplete
                  v-model="newTopic"
                  :items="topics"
                  label="Topic (leave empty to unset)"
                />
              </v-card-text>
              <v-card-actions>
                <v-btn color="blue darken-1" text @click="saveTopic">
                  {{ $t("views.editor.changeTopic.button") }}
                </v-btn>
              </v-card-actions>
            </div>
            <div v-show="currentTab === TABS.MUSIC">
              <VideoEditSongs
                id="musicEditor"
                ref="musicEditor"
                :video="video"
                :current-time="currentTime"
                @timeJump="seekTo"
              />
            </div>
            <div v-if="currentTab === TABS.MENTIONS">
              <VideoEditMentions :video="video" />
            </div>
          </v-col>
          <v-col cols="12">
            <WatchInfo key="info" :video="video" />
          </v-col>
        </v-row>
      </v-col>
    </v-row>
  </v-container>
  <LoadingOverlay v-else :is-loading="isLoading" :show-error="hasError" />
</template>

<script lang="ts">
import Vue from "vue";
import VueYoutube from "@/external/vue-youtube";

import LoadingOverlay from "@/components/common/LoadingOverlay.vue";
import WatchInfo from "@/components/watch/WatchInfo.vue";
import WatchFrame from "@/components/watch/WatchFrame.vue";
import WatchComments from "@/components/watch/WatchComments.vue";
import VideoEditSongs from "@/components/edit/VideoEditSongs.vue";
import VideoEditMentions from "@/components/edit/VideoEditMentions.vue";
import { decodeHTMLEntities } from "@/utils/functions";
// import { dayjs } from "@/utils/time";
import * as icons from "@/utils/icons";
import api from "@/utils/backend-api";

export default {
    name: "Watch",
    metaInfo() {
        return {
            title: this.title,
        };
    },
    components: {
        LoadingOverlay,
        WatchInfo,
        WatchFrame,
        VideoEditSongs,
        WatchComments,
        VideoEditMentions,
    },
    data() {
        return {
            isLoading: true,
            hasError: false,
            id: 0,
            video: null,
            startTime: 0,
            icons,
            currentTab: 0,
            TABS: Object.freeze({
                TOPIC: 0,
                MUSIC: 1,
                MENTIONS: 2,
                SOURCES_CLIPS: 3,
            }),

            newTopic: null,
            topics: [],

            timer: null,
            currentTime: 0,
        };
    },
    computed: {
        videoId() {
            return this.$route.params.id || this.$route.query.v;
        },
        timeOffset() {
            return +this.$route.query.t || this.startTime;
        },
        title() {
            return (this.video && this.video.title && decodeHTMLEntities(this.video.title)) || "";
        },
        role() {
            return this.$store.state.userdata?.user?.role;
        },
    },
    watch: {
        // eslint-disable-next-line func-names
        "$route.params.id": function () {
            this.init();
        },
        // eslint-disable-next-line func-names
        "$route.query.v": function () {
            this.init();
        },
        currentTab() {
            this.initTab();
        },
    },
    created() {
        Vue.use(VueYoutube);
    },
    mounted() {
        // Load specific tab if defined in the tab param
        if (this.$route.params.tab) {
            this.currentTab = this.TABS[this.$route.params.tab.toUpperCase()] || 0;
        }
        this.init();
    },
    beforeDestroy() {
        clearInterval(this.timer);
    },
    methods: {
        init() {
            this.id = this.videoId;
            this.fetchVideo();
            this.initTab();
        },
        initTab() {
            if (this.currentTab === this.TABS.TOPIC) {
                this.populateTopics();
            }
        },
        ready(event) {
            this.player = event;
            this.setTimer();
        },
        seekTo(time, playNow, updateStartTime) {
            if (!this.player) return;
            this.player.seekTo(time);
            if (playNow) this.player.playVideo();
            if (updateStartTime && this.currentTab === this.TABS.MUSIC) {
                this.$refs.musicEditor && this.$refs.musicEditor.setStartTime(time);
                // document.getElementById("musicEditor").scrollIntoView();
            }
        },
        fetchVideo() {
            if (!this.id) throw new Error("Invalid id");
            this.isLoading = true;
            return api
                .video(this.id, null, 1)
                .then(({ data }) => {
                    this.video = data;
                    this.isLoading = false;
                })
                .catch((e) => {
                    this.hasError = true;
                    console.error(e);
                });
        },
        async populateTopics() {
            this.topics = (await api.topics()).data.map((topic) => ({
                value: topic.id,
                text: `${topic.id} (${topic.count ?? 0})`,
            }));
        },
        saveTopic() {
            this.dialog = false;
            api.topicSet(this.newTopic, this.videoId, this.$store.state.userdata.jwt);
            this.topic = this.newTopic;
        },
        setTimer() {
            if (this.timer) clearInterval(this.timer);
            if (this.player) {
                this.timer = setInterval(() => {
                    this.currentTime = this.player.getCurrentTime();
                }, 1000);
            }
        },
    },
};
</script>

<style>
.video-editor .comment-scroller {
    height: 400px;
    height: 60vh;
    overflow: hidden auto;
}

.video-editor .watch-card {
    border: none !important;
    box-shadow: none !important;
}

.video-editor .thumbnail-overlay {
    background-color: rgba(0, 0, 0, 0.5);
    width: 100%;
    height: 100%;
    position: absolute;
    top: 0;
}

.video-editor .thumbnail {
    position: relative;
}
.video-editor .watch-btn-group > .v-btn {
    margin-right: 5px;
}
</style>
