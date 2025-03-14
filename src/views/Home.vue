<template>
  <v-container
    v-touch="{
      right: () => {
        tab = Math.max(tab - 1, 0);
        changeTab(false);
      },
      left: () => {
        tab = Math.min(tab + 1, 2);
        changeTab(false);
      },
    }"
    fluid
    style="min-height: 100%"
    class="d-flex flex-column pt-0"
  >
    <!-- Teleport tabs to nav extension slot -->
    <portal to="mainNavExt" :disabled="!$vuetify.breakpoint.xs || !isActive">
      <v-tabs
        v-model="tab"
        :centered="$vuetify.breakpoint.xs"
        :class="$store.state.settings.darkMode ? 'secondary darken-1' : 'primary lighten-1'"
        :active-class="
          $store.state.settings.darkMode ? 'primary--text text--lighten-3' : 'primary--text text--darken-2'
        "
        @change="changeTab(false)"
      >
        <v-tab class="pa-2">
          {{ liveUpcomingHeaderSplit[1] }}
          <span class="stream-count-chip mx-1 rounded-md primary white--text rounded-lg pa-1">
            {{ lives.length }}
          </span>
          {{ liveUpcomingHeaderSplit[2] }}
          <span class="stream-count-chip ml-1 rounded-md primary white--text rounded-lg pa-1">
            {{ upcoming.length }}
          </span>
        </v-tab>
        <v-tab class="pa-2">
          {{ $t("views.home.recentVideoToggles.official") }}
        </v-tab>
        <v-tab class="pa-2">
          {{ $t("views.home.recentVideoToggles.subber") }}
        </v-tab>
      </v-tabs>
    </portal>
    <LoadingOverlay :is-loading="false" :show-error="hasError" />
    <div v-show="!hasError">
      <template v-if="tab === Tabs.LIVE_UPCOMING">
        <SkeletonCardList v-if="isLoading" :cols="colSizes" :dense="currentGridSize > 0" />
        <div v-if="lives.length || upcoming.length">
          <VideoCardList
            :videos="lives"
            include-channel
            :include-avatar="shouldIncludeAvatar"
            :cols="colSizes"
            :dense="currentGridSize > 0"
            hide-ignored-topics
          />
          <v-divider v-if="lives.length" class="my-3 secondary" />
          <VideoCardList
            :videos="upcoming"
            include-channel
            :include-avatar="shouldIncludeAvatar"
            :cols="colSizes"
            :dense="currentGridSize > 0"
            hide-ignored-topics
          />
        </div>
        <div v-show="!isLoading && lives.length == 0 && upcoming.length == 0" class="ma-auto pa-5 text-center">
          {{ $t("views.home.noStreams") }}
        </div>
      </template>
      <template v-else>
        <keep-alive>
          <generic-list-loader
            v-slot="{ data, isLoading }"
            :key="'vl-home-' + tab + identifier"
            :infinite-load="scrollMode"
            :paginate="!scrollMode"
            :per-page="pageLength"
            :load-fn="getLoadFn()"
          >
            <!-- only keep VideoCardList rendered if scrollMode OR it's not loading. -->
            <VideoCardList
              v-show="scrollMode || !isLoading"
              :videos="data"
              include-channel
              :cols="colSizes"
              :dense="currentGridSize > 0"
              hide-ignored-topics
            />
            <!-- only show SkeletonCardList if it's loading -->
            <SkeletonCardList v-if="isLoading" :cols="colSizes" :dense="currentGridSize > 0" />
          </generic-list-loader>
        </keep-alive>
      </template>
    </div>
  </v-container>
</template>

<script lang="ts">
import VideoCardList from "@/components/video/VideoCardList.vue";
import LoadingOverlay from "@/components/common/LoadingOverlay.vue";
import { mapState } from "vuex";
import reloadable from "@/mixins/reloadable";
import backendApi from "@/utils/backend-api";
import GenericListLoader from "@/components/video/GenericListLoader.vue";
import SkeletonCardList from "@/components/video/SkeletonCardList.vue";
import isActive from "@/mixins/isActive";

export default {
    name: "Home",
    metaInfo() {
        return {
            get title() {
                return "Holodex";
            },
        };
    },
    components: {
        VideoCardList,
        LoadingOverlay,
        GenericListLoader,
        SkeletonCardList,
    },
    mixins: [reloadable, isActive],
    data() {
        return {
            identifier: Date.now(),
            pageLength: 24,
            tab: 0,
            Tabs: Object.freeze({
                LIVE_UPCOMING: 0,
                // ALL: 1,
                ARCHIVE: 1,
                CLIPS: 2,
            }),
            refreshTimer: null,
        };
    },
    computed: {
        ...mapState("home", ["live", "isLoading", "hasError"]),
        scrollMode() {
            return this.$store.state.settings.scrollMode;
        },
        currentGridSize: {
            get() {
                return this.$store.state.currentGridSize;
            },
            set(val) {
                this.$store.commit("setCurrentGridSize", val);
            },
        },
        colSizes() {
            return {
                xs: 1 + this.currentGridSize,
                sm: 2 + this.currentGridSize,
                md: 3 + this.currentGridSize,
                lg: 4 + this.currentGridSize,
                xl: 5 + this.currentGridSize,
            };
        },
        shouldIncludeAvatar() {
            if (this.$vuetify.breakpoint.md && this.currentGridSize > 1) return false;
            if (this.$vuetify.breakpoint.sm && this.currentGridSize > 0) return false;
            if (this.$vuetify.breakpoint.xs && this.currentGridSize > 0) return false;
            return true;
        },
        lives() {
            return this.live.filter((v) => v.status === "live");
        },
        upcoming() {
            return this.live.filter((v) => v.status === "upcoming");
        },
        liveUpcomingHeaderSplit() {
            return this.$t("views.home.liveOrUpcomingHeading").match(/(.+)([\\/／・].+)/);
        },
    },
    watch: {
        // eslint-disable-next-line func-names
        "$store.state.currentOrg": function () {
            this.init();
        },
        // eslint-disable-next-line func-names
        "$store.state.visibilityState": function () {
            if (this.isActive && this.$store.state.visibilityState === "visible") {
                this.$store.dispatch("home/fetchLive", { force: false });
            }
        },
        tab() {
            // Scroll to top
            this.$nextTick(() => {
                window.scrollTo(0, 0);
            });
            this.changeTab();
        },
    },
    created() {
        this.init();
        this.setAutoRefresh();
    },
    activated() {
        this.changeTab(true);
        this.setAutoRefresh();
    },
    deactivated() {
        if (this.refreshTimer) {
            clearInterval(this.refreshTimer);
            this.refreshTimer = null;
        }
    },
    beforeDestroy() {
        if (this.refreshTimer) {
            clearInterval(this.refreshTimer);
            this.refreshTimer = null;
        }
    },
    methods: {
        setAutoRefresh() {
            if (this.refreshTimer) clearInterval(this.refreshTimer);
            this.refreshTimer = setInterval(() => {
                this.$store.dispatch("home/fetchLive", { force: false });
            }, 2 * 60 * 1000);
        },
        changeTab(preservePage = true) {
            // Sync the hash to current tab
            const toHash = {
                0: "",
                1: "archive",
                2: "clips",
            };
            this.$router
                .replace({
                    // set page to 0 if on scroll mode
                    query: preservePage && {
                        ...this.$route.query,
                    },
                    hash: toHash[this.tab] || "",
                })
                .catch(() => {
                    // Navigation duplication error expected, catch it and move on
                });
        },
        init() {
            this.$store.commit("home/resetState");
            this.$store.dispatch("home/fetchLive", { force: true });
            this.identifier = Date.now();
            switch (this.$route.hash) {
                case "#archive":
                    this.tab = this.Tabs.ARCHIVE;
                    break;
                case "#clips":
                    this.tab = this.Tabs.CLIPS;
                    break;
                default:
                    this.tab = this.Tabs.LIVE_UPCOMING;
                    break;
            }
        },
        // called from mixin, simulate reload
        reload() {
            this.init();
        },
        getLoadFn() {
            return async (offset, limit) => {
                const res = await backendApi.videos({
                    status: "past",
                    ...{ type: this.tab === this.Tabs.ARCHIVE ? "stream" : "clip" },
                    include: "clips",
                    org: this.$store.state.currentOrg.name,
                    lang: this.$store.state.settings.clipLangs.join(","),
                    paginated: !this.scrollMode,
                    limit,
                    offset,
                });
                return res.data;
            };
        },
    },
};
</script>
<style>
.v-slide-group__prev--disabled {
    display: none !important;
}
/* shared with favorites.vue */
.stream-count-chip {
    letter-spacing: normal;
    min-width: 24px;
}
</style>
