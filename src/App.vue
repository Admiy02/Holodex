<template>
  <v-app
    :style="{ background: $vuetify.theme.themes[darkMode ? 'dark' : 'light'].background }"
    :class="{ 'bump-bottom': $store.state.music.isOpen }"
  >
    <portal-target name="music-playback-background" />

    <MainNav />

    <v-main style="transition: none">
      <PullToRefresh />
      <keep-alive max="4" exclude="Watch,MugenClips,EditVideo,MultiView,Channel,Playlists,About">
        <router-view :key="viewKey" />
      </keep-alive>
    </v-main>

    <v-snackbar
      v-if="needRefresh"
      bottom
      right
      :value="needRefresh"
      :timeout="-1"
      color="primary"
    >
      {{ $t("views.app.update_available") }}
      <template #action>
        <v-btn text class="ml-auto" @click="updateServiceWorker">
          {{ $t("views.app.update_btn") }}
        </v-btn>
        <v-btn text class="ml-auto" @click="needRefresh = false">
          {{ $t("views.app.close_btn") }}
        </v-btn>
      </template>
    </v-snackbar>
    <v-snackbar
      v-if="showUpdateDetails"
      bottom
      center
      :value="showUpdateDetails"
      color="primary"
      :timeout="-1"
    >
      {{ $t("views.app.check_about_page") }}
      <template #action>
        <v-btn
          text
          class="ml-auto"
          to="/about#changelog"
          @click="showUpdateDetails = false"
        >
          Changelog
        </v-btn>
        <v-btn text class="ml-auto" @click="showUpdateDetails = false">
          {{ $t("views.app.close_btn") }}
        </v-btn>
      </template>
    </v-snackbar>
    <ReportDialog />
  </v-app>
</template>

<script lang="ts">
import MainNav from "@/components/nav/MainNav.vue";
import ReportDialog from "@/components/common/ReportDialog.vue";
import PullToRefresh from "@/components/common/PullToRefresh.vue";
import { loadLanguageAsync } from "./plugins/vuetify";
import { axiosInstance } from "./utils/backend-api";
import * as SW from "./sw";

export default {
    name: "App",
    // default meta info
    metaInfo: {
        title: "Holodex",
    },
    components: {
        MainNav,
        ReportDialog,
        PullToRefresh,
    },
    data() {
        return {
            needRefresh: false,
            favoritesUpdateTask: null,
        };
    },
    computed: {
        viewKey() {
            const key = this.$route.path;
            // channel has subviewws that will cause unwanted keep-alive instances
            // Key them all under channel/:id to avoid duplicating
            if (key.match("^/channel/.{16}")) {
                return key.substring(0, 33);
            }
            return key;
        },
        darkMode() {
            return this.$store.state.settings.darkMode;
        },
        showUpdateDetails: {
            set(val) {
                this.$store.commit("setShowUpdatesDetail", val);
            },
            get() {
                return this.$store.state.showUpdateDetails;
            },
        },
        lang() {
            // connected to the watch.lang hook below.
            return this.$store.state.settings.lang;
        },
    },
    watch: {
        darkMode() {
            this.$vuetify.theme.dark = this.darkMode;
        },
        lang() {
            // watches the computed.lang variable and updates vue I18N
            // import(`dayjs/locale/${this.lang}`) // ES 2015
            loadLanguageAsync(this.$store.state.settings.lang);
            // setDayjsLang(this.$store.state.settings.lang);
        },
        // watches change in breakpoint from vuetify and updates store
        // eslint-disable-next-line func-names
        "$vuetify.breakpoint.name": function () {
            this.updateIsMobile();
        },
        // eslint-disable-next-line func-names
        "$store.state.visibilityState": function () {
            if (this.$store.state.visibilityState === "visible") {
                this.$store.dispatch("favorites/fetchLive", { force: false, minutes: 5 });
            }
        },
    },
    async created() {
        this.$store.commit("setVisiblityState", document.visibilityState);
        document.addEventListener("visibilitychange", () => {
            this.$store.commit("setVisiblityState", document.visibilityState);
        });
        axiosInstance.interceptors.response.use(undefined, (error) => {
            // Any status codes that falls outside the range of 2xx cause this function to trigger
            // Do something with response error
            if (error.response) {
                // The request was made and the server responded with a status code
                // that falls out of the range of 2xx
                this.$gtag.exception({
                    description: `${error.response.config.method} ${error.response.config.url}->${error.response.status}`,
                    fatal: true,
                });
                this.$gtag.event(`xhr:${error.response.status}`, {
                    event_category: "xhrError",
                    event_label: `${error.response.config.method} ${error.response.config.url} -> ${error.response.status}`,
                });
                console.error(error.response.data);
                console.error(error.response.status);
                console.error(error.response.headers);
            } else if (error.request) {
                // The request was made but no response was received
                // `error.request` is an instance of XMLHttpRequest in the browser and an instance of
                // http.ClientRequest in node.js
                this.$gtag.exception({
                    description: "No Response Received",
                    fatal: true,
                });
                console.error(error.request);
            } else {
                // Something happened in setting up the request that triggered an Error
                this.$gtag.exception({
                    description: `Generic: ${error.message}`,
                    fatal: true,
                });
                console.error("Error", error.message);
            }
            return Promise.reject(error);
        });

        // set theme
        this.$vuetify.theme.dark = this.darkMode;
        // set lang
        // setDayjsLang(this.$store.state.settings.lang);
        this.$i18n.locale = this.$store.state.settings.lang;
        this.$vuetify.lang.current = this.$store.state.settings.lang;

        // relog if necessary:
        this.$store.dispatch("loginCheck");

        if (this.favoritesUpdateTask) clearInterval(this.favoritesUpdateTask);

        this.favoritesUpdateTask = setInterval(() => {
            this.$store.dispatch("favorites/fetchLive", { minutes: 5 });
        }, 6 * 60 * 1000);

        // queue up a favorite updating task to run in 5 seconds.
        // rationale: if I'm reloading the page i expect to have recent Favorites data in my side bar and stuff.
        // why 5 seconds? 1. it's to give the illusion of the page updating quickly.
        // 2. it's so if you're landing on the /favorites page this request occurs AFTER that one's to save bandwidth.
        setTimeout(() => {
            this.$store.dispatch("favorites/fetchLive", { force: false, minutes: 2 });
        }, 5000);
        // check current breakpoint and set isMobile
        this.updateIsMobile();

        SW.setNeedsRefreshCallback(() => {
            console.log("sw needs refresh");
            this.needRefresh = true;
        });
    },
    beforeDestroy() {
        if (this.favoritesUpdateTask) clearInterval(this.favoritesUpdateTask);
    },
    methods: {
        updateIsMobile() {
            this.$store.commit("setIsMobile", ["xs", "sm"].includes(this.$vuetify.breakpoint.name));
        },
        async updateServiceWorker() {
            console.log("update service worker");
            await SW.updateServiceWorker();
            this.showUpdateDetails = true;
            window.location.reload();
        },
    },
};
</script>
<style>
.no-decoration {
    text-decoration: none;
    color: inherit !important;
}

html {
    overflow-y: auto;
}

body {
    overscroll-behavior-y: contain;
    background: black;
    padding-left: min(calc(env(safe-area-inset-left)), 30px);
    padding-right: min(calc(env(safe-area-inset-right)), 30px);
}
.row {
    margin: 0px -12px;
}
.bump-bottom .v-main__wrap {
    /* a bit of janky bottom spacing to allow all clients to scroll to bottom */
    padding-bottom: 140px;
}

/* pull to refresh skin */

.ptr--ptr {
    box-shadow: none !important;
}

.ptr--box {
    padding: 0px !important;
    justify-content: center;
    display: flex;
}

/* icon size */
.ptr--icon,
.ptr--text > svg {
    width: 32px;
    height: 32px;
}

/* rotate left arrow to be down arrow, micro bandwidth savings */
.ptr--icon {
    transform: rotate(90deg);
}

/* only display either icon or text */
.ptr--ptr.ptr--refresh .ptr--content .ptr--icon {
    display: none;
}

.ptr--text {
    display: none;
}

/* rotate arrow when threshold reached */
.ptr--ptr.ptr--release .ptr--content .ptr--icon {
    transform: rotate(270deg);
}

/* show text with refresh spinner and animate */
.ptr--ptr.ptr--refresh .ptr--content .ptr--text {
    animation: spin 1.1s infinite linear;
    display: block;
}

@keyframes spin {
    0% {
        -webkit-transform: rotate(0deg);
        transform: rotate(0deg);
    }
    100% {
        -webkit-transform: rotate(360deg);
        transform: rotate(360deg);
    }
}
.thin-scroll-bar {
    scrollbar-width: thin;
}

.thin-scroll-bar::-webkit-scrollbar {
    width: 8px;
    height: 8px;
}

.thin-scroll-bar::-webkit-scrollbar-track {
    background: rgba(0, 0, 0, 0.2);
}

.thin-scroll-bar::-webkit-scrollbar-thumb {
    background: rgba(182, 182, 182, 0.4);
}
</style>
