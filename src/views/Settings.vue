<template>
  <v-container :style="slim && 'width: 500px;'" :class="{ 'pa-0': slim }">
    <v-row dense>
      <v-col v-if="!slim" cols="12">
        <div class="text-h4 mb-4">
          {{ $t("views.settings.title") }}
        </div>
      </v-col>
      <v-col :cols="slim ? 12 : currentCol">
        <v-sheet class="settings-group">
          <v-card-title class="py-1">
            <v-icon
              large
              disabled
              left
              class="ml-n3"
            >
              {{ mdiEarth }}
            </v-icon>

            <span class="text-h6 font-weight-light">{{ $t("views.settings.languageSettings") }}</span>
          </v-card-title>
          <v-card-text>
            <v-select
              v-model="language"
              :items="langs"
              item-value="val"
              :prepend-icon="icons.mdiTranslate"
            >
              <template #item="{ item }">
                <!-- {{item}} -->
                <div>
                  <span class="primary--text" style="">{{ item.display }}</span>
                  <span class="px-2 text--secondary text-caption"> ♡ {{ item.credit }}</span>
                </div>
              </template>
              <template #selection="{ item }">
                <span class="primary--text" style="">{{ item.display }}</span>
              </template>
            </v-select>
            <v-switch
              v-model="useEnName"
              class="v-input--reverse v-input--expand mt-0"
              inset
              prepend-icon=" "
              :label="$t('views.settings.useEnglishNameLabel')"
              :messages="$t('views.settings.useEnglishNameMsg')"
            />

            <div class="mt-6 mb-4">
              <v-icon style="margin-right: 9px">
                {{ mdiFilter }}
              </v-icon>
              <span class="text-body-1">{{ $t("views.settings.clipLanguageSelection") }}</span>
            </div>
            <!-- <v-container fluid> -->
            <v-checkbox
              v-for="l in TL_LANGS"
              :key="l.value + 'settingcheckbox'"
              v-model="clipLangs"
              :label="l.text"
              :value="l.value"
              dense
              class="ml-4 mt-n2"
            />
            <!-- </v-container> -->
          </v-card-text>
        </v-sheet>
      </v-col>
      <v-col :cols="slim ? 12 : currentCol">
        <v-sheet class="settings-group">
          <v-card-title class="py-1">
            <v-icon
              large
              disabled
              left
              class="ml-n3"
            >
              {{ icons.mdiCog }}
            </v-icon>
            <span class="text-h6 font-weight-light">{{ $t("views.settings.siteNavigationSettings") }}</span>
          </v-card-title>
          <v-card-text>
            <v-switch
              v-model="darkMode"
              class="v-input--reverse v-input--expand"
              :label="$t('views.settings.darkModeLabel')"
              hide-details
              inset
              :prepend-icon="mdiWeatherNight"
            />
            <!-- :messages="$t('views.settings.darkModeMsg')" -->
            <div class="mt-6">
              <v-icon style="margin-right: 9px">
                {{ mdiPalette }}
              </v-icon>
              <span class="text-body-1">{{ $t("views.settings.theme") }}</span>
              <!-- <div class="theme-preview d-inline float-right text-body-1">
                                <span :style="`background:${themeSet[themeId].themes[mode].primary}`"></span>
                                <span :style="`background:${themeSet[themeId].themes[mode].secondary}`"></span>
                                {{ themeSet[themeId].name }}
                            </div> -->

              <v-select
                v-model="themeId"
                class="mt-0 d-inline-block float-right"
                hide-details
                dense
                style="width: 150px"
                :items="themeSet"
                item-value="id"
              >
                <template #item="{ item }">
                  <div class="theme-preview">
                    <span :style="`background:${item.themes[mode].primary}`" />
                    <span :style="`background:${item.themes[mode].secondary}`" />
                    {{ item.name }}
                  </div>
                </template>
                <template #selection="{ item }">
                  <div class="theme-preview">
                    <span :style="`background:${item.themes[mode].primary}`" />
                    <span :style="`background:${item.themes[mode].secondary}`" />
                    {{ item.name }}
                  </div>
                </template>
              </v-select>
            </div>

            <div class="mb-0 mt-6">
              <v-icon style="margin-right: 9px">
                {{ mdiHome }}
              </v-icon>
              <span class="text-body-1">{{ $t("views.settings.defaultPage") }}</span>
            </div>
            <v-select
              v-model="defaultOpen"
              class="mt-n4"
              prepend-icon=" "
              :items="defaultOpenChoices"
              :messages="$t('views.settings.defaultPageMsg')"
            />

            <div class="mb-0 mt-6">
              <v-icon style="margin-right: 9px">
                {{ icons.mdiGrid }}
              </v-icon>
              <span class="text-body-1">{{ $t("views.settings.gridSizeLabel") }}</span>
            </div>
            <v-select
              v-model="currentGridSize"
              prepend-icon=" "
              class="mt-n4"
              :items="[
                { text: $t('views.settings.gridSize[0]'), value: 0 },
                { text: $t('views.settings.gridSize[1]'), value: 1 },
                { text: $t('views.settings.gridSize[2]'), value: 2 },
              ]"
              :messages="$t('views.settings.gridSizeMsg')"
            />
            <v-switch
              v-model="scrollMode"
              class="v-input--reverse v-input--expand mt-6"
              :prepend-icon="mdiBookOpenPageVariantOutline"
              inset
              :label="$t('views.settings.scrollModeLabel')"
              :messages="$t('views.settings.scrollModeMsg')"
            />
            <v-switch
              v-model="redirectMode"
              class="v-input--reverse v-input--expand mt-6"
              :prepend-icon="icons.mdiYoutube"
              inset
              :label="$t('views.settings.redirectModeLabel')"
              :messages="$t('views.settings.redirectModeMsg')"
            />
          </v-card-text>
        </v-sheet>
      </v-col>
      <v-col v-if="!slim">
        <v-sheet class="settings-group">
          <v-card-title class="py-1">
            <v-icon
              large
              disabled
              left
              class="ml-n3"
            >
              {{ icons.mdiAnimationPlay }}
            </v-icon>
            <span class="text-h6 font-weight-light">{{ $t("views.settings.videoFeedSettings") }}</span>
          </v-card-title>
          <v-card-text>
            <v-switch
              v-model="hideCollabStreams"
              :prepend-icon="mdiEyeOff"
              class="v-input--reverse v-input--expand"
              inset
              :label="$t('views.settings.hideCollabStreamsLabel')"
              :messages="$t('views.settings.hideCollabStreamsMsg')"
            />

            <v-autocomplete
              v-model="ignoredTopics"
              :items="topics"
              prepend-icon=" "
              multiple
              chips
              clearable
              deletable-chips
              :label="$t('views.settings.ignoredTopicsLabel')"
              :hint="$t('views.settings.ignoredTopicsMsg')"
              persistent-hint
            />

            <v-switch
              v-model="hideThumbnail"
              prepend-icon=" "
              class="v-input--reverse v-input--expand"
              inset
              :label="$t('views.settings.hideVideoThumbnailsLabel')"
              :messages="$t('views.settings.hideVideoThumbnailsMsg')"
            />
          </v-card-text>
        </v-sheet>
        <br v-if="!slim">
        <v-btn
          v-if="!slim"
          block
          color="warning"
          @click="resetSettings"
        >
          {{ $t("views.settings.resetAllSettings") }}
        </v-btn>
      </v-col>
    </v-row>
  </v-container>
</template>

<script lang="ts">
import { langs } from "@/plugins/vuetify";
import {
    mdiFilter,
    mdiEarth,
    mdiPalette,
    mdiCogBox,
    mdiWeatherNight,
    mdiHome,
    mdiEyeOff,
    mdiBookOpenPageVariantOutline,
} from "@mdi/js";
import { TL_LANGS } from "@/utils/consts";
import themeSet from "@/utils/themes";
import { syncState } from "@/utils/functions";
import Vue from "vue";
import backendApi from "@/utils/backend-api";

export default {
    name: "Settings",
    metaInfo() {
        const vm = this;
        return {
            get title() {
                return `${vm.$t("component.mainNav.settings")} - Holodex`;
            },
        };
    },
    props: {
        slim: {
            type: Boolean,
            default: false,
        },
    },
    data() {
        return {
            langs,
            mdiFilter,
            mdiEarth,
            mdiPalette,
            mdiEyeOff,
            mdiCogBox,
            mdiWeatherNight,
            mdiHome,
            mdiBookOpenPageVariantOutline,
            TL_LANGS,

            themeId: +localStorage.getItem("theme") || 0,
            themeSet,
            defaultOpenChoices: Object.freeze([
                {
                    text: this.$t("component.mainNav.home"),
                    value: "home",
                },
                {
                    text: this.$t("component.mainNav.favorites"),
                    value: "favorites",
                },
                {
                    text: this.$t("component.mainNav.multiview"),
                    value: "multiview",
                },
            ]),
            topics: [],
        };
    },
    computed: {
        ...syncState("settings", [
            "darkMode",
            "redirectMode",
            "autoplayVideo",
            "scrollMode",
            "hideThumbnail",
            "defaultOpen",
            "hideCollabStreams",
            "ignoredTopics",
        ]),
        currentCol() {
            if (this.$vuetify.breakpoint.smAndDown) return 12;
            if (this.$vuetify.breakpoint.md) return 6;
            if (this.$vuetify.breakpoint.lgAndUp) return 4;
            return null;
        },
        currentGridSize: {
            get() {
                return this.$store.state.currentGridSize;
            },
            set(val) {
                this.$store.commit("setCurrentGridSize", val);
            },
        },
        useEnName: {
            get() {
                return this.$store.getters["settings/useEnName"];
            },
            set(val) {
                this.$store.commit("settings/setUseEnName", val);
            },
        },
        language: {
            get() {
                return this.$store.state.settings.lang;
            },
            set(val) {
                this.$store.commit("settings/setLanguage", val);
            },
        },
        clipLangs: {
            get() {
                return this.$store.state.settings.clipLangs;
            },
            set(val: any[]) {
                // sort array to increase cache hit rate
                this.$store.commit("settings/setClipLangs", val.sort());
            },
        },
        ignoredTopics: {
            get() {
                return this.$store.state.settings.ignoredTopics;
            },
            set(val: any[]) {
                this.$store.commit("settings/setIgnoredTopics", val.sort());
            },
        },
        theme() {
            return this.$vuetify.theme;
        },
        mode() {
            return this.darkMode ? "dark" : "light";
        },
    },
    watch: {
        hideCollabStreams() {
            this.$store.commit("favorites/setLastLiveUpdate", 0);
        },
        themeId(nw) {
            localStorage.setItem("theme", `${nw}`);
            Vue.set(this.$vuetify.theme.themes, "dark", {
                ...this.$vuetify.theme.themes.dark,
                ...themeSet[nw].themes.dark,
            });
            Vue.set(this.$vuetify.theme.themes, "light", {
                ...this.$vuetify.theme.themes.light,
                ...themeSet[nw].themes.light,
            });
        },
    },
    async mounted() {
        backendApi.topics().then(({ data }) => {
            this.topics = data.map(({ id, count }) => ({ value: id, text: `${id} (${count})` }));
        });
    },
    methods: {
        resetSettings() {
            // eslint-disable-next-line no-restricted-globals,no-alert
            if (confirm(this.$t("views.settings.resetAllSettingsWarning"))) {
                this.$store.commit("resetState");
                this.$store.commit("home/resetState");
                this.$store.commit("channel/resetState");
                this.$store.commit("channels/resetState");
                this.$store.commit("watch/resetState");
                this.$store.commit("settings/resetState");
                this.$store.commit("favorites/resetState");
                this.$store.commit("music/resetState");
                this.$store.commit("multiview/resetState");
                this.$store.commit("playlist/resetState");
                this.$store.commit("history/resetState");
                window.location.reload();
            }
        },
    },
};
</script>

<style lang="scss">
.settings-group {
    padding: 12px;
    border: 1px solid var(--v-primary-base);
    border-radius: 8px;
}
.theme-preview span {
    width: 1rem;
    height: 0.5rem;
    margin-bottom: 0.5rem;
    display: inline-block;
    border-radius: 3px 3px 0 0;
    margin-left: 2px;
    &:nth-child(2) {
        margin-left: -1rem;
        height: 0.5rem;
        margin-bottom: 0rem;
        border-radius: 0 0 3px 3px;
    }
}

.v-input--reverse .v-input__slot {
    flex-direction: row-reverse;
    justify-content: flex-end;
    .v-application--is-ltr & {
        .v-input--selection-controls__input {
            margin-right: 0;
            margin-left: 8px;
        }
    }
    .v-application--is-rtl & {
        .v-input--selection-controls__input {
            margin-left: 0;
            margin-right: 8px;
        }
    }
}

// Bonus "expand" variant
.v-input--expand .v-input__slot {
    // justify-content: space-between;
    .v-label {
        display: block;
        flex: 1;
    }
}
</style>
