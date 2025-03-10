<template>
  <v-hover v-slot="{ hover }">
    <v-list-item @click.stop="$emit('play', song)">
      <v-hover v-slot="{ hover: hoverInner }">
        <v-list-item-avatar tile>
          <!-- actual artwork -->
          <v-img v-if="song.art" lazy-src :src="song.art" />
          <!-- artwork not available, have a stand-in -->
          <v-sheet
            v-else
            width="100%"
            height="100%"
            color="grey darken-1"
            class="d-flex pa-1"
          >
            <v-btn
              small
              icon
              class="ma-auto"
              outlined
              disabled
            >
              <v-icon small>
                {{ icons.mdiMusic }}
              </v-icon>
            </v-btn>
          </v-sheet>
          <!-- Queue up button or default item click button -->
          <v-sheet
            v-if="hover && !hoverInner"
            width="100%"
            height="100%"
            color="transparent"
            class="d-flex pa-1 hover-item"
            style="position: absolute; left: 0px"
          >
            <v-btn
              x-small
              fab
              class="ma-auto"
              color="blue lighten-3"
              elevation="2"
            >
              <v-icon small>
                {{ hoverIcon }}
              </v-icon>
            </v-btn>
          </v-sheet>
          <!-- Play immediately button over the artwork -->
          <v-sheet
            v-if="$listeners.playNow && hoverInner"
            width="100%"
            height="100%"
            color="transparent"
            class="d-flex pa-1 hover-art"
            style="position: absolute; left: 0px"
          >
            <v-btn
              x-small
              fab
              class="ma-auto"
              color="blue lighten-3"
              elevation="2"
              @click.stop.prevent="$emit('playNow', song)"
            >
              <v-icon small>
                {{ artworkHoverIcon }}
              </v-icon>
            </v-btn>
          </v-sheet>
        </v-list-item-avatar>
      </v-hover>
      <v-list-item-content class="py-1 pt-1">
        <v-list-item-subtitle class="text--primary text-subtitle-1">
          <a
            v-if="alwaysShowDeletion || (detailed && $listeners.remove && userCanDelete)"
            class="text-caption error--text float-right ml-1 song-clickable"
            @click.stop="$emit('remove', song)"
          >
            {{ $t("component.media.remove") }}
          </a>
          <div v-if="detailed" class="float-right text-caption">
            [{{ secondsToHuman(song.start) }} - {{ secondsToHuman(song.end) }}]
          </div>

          <span class="limit-width">
            {{ song.name }} /
            <span class="primary--text">{{ song.original_artist }}</span>
          </span>
        </v-list-item-subtitle>

        <v-list-item-subtitle class="text--caption">
          <div class="float-right">
            <span v-if="showTime" class="muted">{{ formattedTime }}</span>
            {{ Math.floor((song.end - song.start) / 60) }}:{{
              (Math.round(song.end - song.start) % 60).toString().padStart(2, "0")
            }}
          </div>

          <span v-if="$listeners.channel" class="song-clickable" @click.stop="$emit('channel', song)">
            {{ song.channel[nameProperty] || song.channel.name }}
          </span>
          <span v-else> {{ song.channel[nameProperty] || song.channel.name }} </span>
        </v-list-item-subtitle>
        <!-- Else: -->
      </v-list-item-content>
    </v-list-item>
  </v-hover>
</template>

<script lang="ts">
import { formatDistance, secondsToHuman } from "@/utils/time";

export default {
    name: "SongItem",
    props: {
        song: {
            type: Object,
            required: true,
        },
        showArtist: {
            type: Boolean,
            default: true,
        },
        showSongArt: {
            type: Boolean,
            default: true,
        },
        verticalList: {
            type: Boolean,
            default: true,
        },
        detailed: {
            // will only show remove button if listener exists and user has DELETE/WRITE priviledge to this song.
            type: Boolean,
            default: false,
        },
        alwaysShowDeletion: {
            // set to true to FORCE the remove button to show up. For use in playlists where removal is from playlist.
            type: Boolean,
            default: false,
        },
        showTime: {
            type: Boolean,
            default: false,
        },
        hoverIcon: {
            type: String,
            default: null,
        },
        artworkHoverIcon: {
            type: String,
            default: null,
        },
    },
    data() {
        return {
            albumArt: null,
        };
    },
    computed: {
        userCanDelete() {
            const u = this.$store.state.userdata;
            return (
                u
                && u.user
                && u.user.role
                && u.user.id
                && (u.user.role !== "user" || +u.user.id === +this.song.creator_id)
            );
        },
        formattedTime() {
            return formatDistance(this.song.available_at, this.$store.state.settings.lang, this.$t.bind(this));
        },
        nameProperty() {
            return this.$store.state.settings.nameProperty;
        },
    },
    mounted() {},
    methods: {
        secondsToHuman,
    },
};
</script>

<style scoped>
.limit-width {
    white-space: normal;
    overflow: hidden;
    text-overflow: ellipsis;
    /* https://css-tricks.com/almanac/properties/w/word-break/ */
    word-break: break-all;
    word-break: break-word;

    -webkit-hyphens: auto;
    -moz-hyphens: auto;
    hyphens: auto;

    display: -webkit-box;
    line-clamp: 1;
    -webkit-line-clamp: 1;
    -webkit-box-orient: vertical;

    justify-content: left;
    text-align: left;
}
.song-clickable {
    text-decoration: none;
}
.song-clickable:hover {
    text-decoration: underline;
    background-color: rgba(120, 120, 120, 0.4);
}
.text--caption .muted {
    opacity: 0.4;
}
</style>
