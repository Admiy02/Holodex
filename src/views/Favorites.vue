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
    class="d-flex flex-column"
  >
    <template v-if="isLoggedIn && favorites.length > 0">
      <!-- Teleport tabs to nav extension slot -->
      <portal to="mainNavExt" :disabled="!$vuetify.breakpoint.xs || !isActive">
        <v-tabs
          v-model="tab"
          :centered="$vuetify.breakpoint.xs"
          :class="$store.state.settings.darkMode ? 'secondary darken-1' : 'primary lighten-1'"
          :active-class="
            $store.state.settings.darkMode
              ? 'primary--text text--lighten-3'
              : 'primary--text text--darken-2'
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
              :hide-collabs="shouldHideCollabs"
              hide-ignored-topics
            />
            <v-divider v-if="lives.length" class="my-3 secondary" />
            <VideoCardList
              :videos="upcoming"
              include-channel
              :include-avatar="shouldIncludeAvatar"
              :cols="colSizes"
              :dense="currentGridSize > 0"
              :hide-collabs="shouldHideCollabs"
              hide-ignored-topics
            />
          </div>
          <div
            v-show="!isLoading && lives.length == 0 && upcoming.length == 0"
            class="ma-auto pa-5 text-center"
          >
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
                :hide-collabs="shouldHideCollabs"
                hide-ignored-topics
              />
              <!-- only show SkeletonCardList if it's loading -->
              <SkeletonCardList v-if="isLoading" :cols="colSizes" :dense="currentGridSize > 0" />
            </generic-list-loader>
          </keep-alive>
        </template>
      </div>
    </template>
    <template v-else>
      <div class="ma-auto d-flex flex-column align-center">
        <v-icon color="primary" large>
          {{ icons.mdiHeart }}
        </v-icon>
        <div class="text-body-1 text-center" v-html="$t('views.favorites.promptForAction')" />
        <v-btn :to="isLoggedIn ? '/channel' : '/login'">
          {{ isLoggedIn ? $t("views.favorites.manageFavorites") : $t("component.mainNav.login") }}
        </v-btn>
      </div>
    </template>
  </v-container>
</template>

<script lang="ts">
import VideoCardList from "@/components/video/VideoCardList.vue";
import LoadingOverlay from "@/components/common/LoadingOverlay.vue";
// eslint-disable-next-line no-unused-vars
import * as icons from "@/utils/icons";
import { mapState } from "vuex";
import reloadable from "@/mixins/reloadable";
import isActive from "@/mixins/isActive";
import backendApi from "@/utils/backend-api";
import GenericListLoader from "@/components/video/GenericListLoader.vue";
import SkeletonCardList from "@/components/video/SkeletonCardList.vue";

export default {
    name: "Favorites",
    components: {
        VideoCardList,
        LoadingOverlay,
        GenericListLoader,
        SkeletonCardList,
    },
    mixins: [reloadable, isActive],
    metaInfo() {
        const vm = this;
        return {
            get title() {
                return `${vm.$t("component.mainNav.favorites")} - Holodex`;
            },
        };
    },
    data() {
        return {
            icons,
            identifier: +new Date(),
            pageLength: 24,
            tab: 0,
            Tabs: Object.freeze({
                LIVE_UPCOMING: 0,
                ARCHIVE: 1,
                CLIPS: 2,
            }),
        };
    },
    computed: {
        ...mapState("favorites", ["favorites", "live", "isLoading", "hasError", "currentOffset"]),
        isLoggedIn() {
            return this.$store.getters.isLoggedIn;
        },
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
        shouldHideCollabs() {
            return this.tab !== this.Tabs.CLIPS && this.$store.state.settings.hideCollabStreams;
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
        favoritesSet() {
            return new Set(this.favorites.map((x) => x.id));
        },
    },
    watch: {
        favorites: {
            deep: true,
            handler(nw, old) {
                if (isActive && nw.find((c, index) => old[index] && c.id !== old[index].id)) {
                    this.init(false);
                }
            },
        },
        tab() {
            this.changeTab();
            // Scroll to top
            this.$nextTick(() => {
                window.scrollTo(0, 0);
            });
        },
    },
    created() {
        this.init(true);
    },
    activated() {
        this.changeTab(true);
    },
    methods: {
        init(updateFavorites) {
            if (this.favorites.length > 0 && this.isLoggedIn) {
                if (updateFavorites) this.$store.dispatch("favorites/fetchFavorites");
                this.$store.dispatch("favorites/fetchLive", { force: true, minutes: 2 });
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
            }
        },
        reload() {
            this.init();
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
        getLoadFn() {
            return async (offset, limit) => {
                const res = await backendApi
                    .favoritesVideos(this.$store.state.userdata.jwt, {
                        status: "past",
                        ...{ type: this.tab === this.Tabs.ARCHIVE ? "stream" : "clip" },
                        include: "clips",
                        lang: this.$store.state.settings.clipLangs.join(","),
                        paginated: !this.scrollMode,
                        limit,
                        offset,
                    })
                    .catch((err) => {
                        console.error(err);
                        this.$store.dispatch("loginVerify"); // check if the user is actually logged in.
                        throw err;
                    });
                return res.data;
            };
        },
    },
};
</script>
