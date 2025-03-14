<template>
  <div class="layout-preview" :class="{ mobile: mobile, desktop: !mobile, 'theme--light': !$vuetify.theme.dark }">
    <template v-for="l in layout">
      <div :key="l.i" class="cell" :style="getStyle(l)">
        <span v-if="content && content[l.i] && content[l.i].type === 'chat'">💬</span>
      </div>
    </template>
  </div>
</template>

<script lang="ts">
export default {
    name: "LayoutPreview",
    props: {
        layout: {
            type: Array,
            required: true,
        },
        content: {
            type: Object,
            required: false,
        },
        mobile: {
            type: Boolean,
            default: false,
        },
    },
    methods: {
        getStyle(l) {
            // viewport is constricted to 24 cols and 24 rows
            // assuming nothing is off the screen, 0 < x < 24, 0 < y < 24
            // x/24 * 100 = percentage of the div
            function px(num) {
                return `${num * (100 / 24)}%`;
            }
            return {
                top: px(l.y),
                left: px(l.x),
                width: px(l.w),
                height: px(l.h),
                ...(this.content && this.content[l.i] && this.content[l.i].type === "chat"
                    ? { "background-color": `${this.$vuetify.theme.parsedTheme.warning.base}44` }
                    : { "background-color": `${this.$vuetify.theme.parsedTheme.info.base}44` }),
            };
        },
    },
};
</script>

<style scoped>
.layout-preview {
    display: inline-block;
    border: 2px solid #424242;
    background-color: #424242;
}

.cell {
    position: absolute;
    border: 2px solid #424242;
    background-color: #9e9e9e;
    box-sizing: border-box;
    display: flex;
    justify-content: center;
    align-items: center;
}

.layout-preview.theme--light > .cell > span {
    color: black;
}

.layout-preview.theme--light {
    border-color: #f5f5f5;
    background-color: #f5f5f5;
}
.layout-preview.theme--light > .cell {
    border-color: #f5f5f5;
    background-color: #e0e0e0;
}

.desktop.layout-preview {
    width: 192px;
    height: 108px;
    position: relative;
    overflow: hidden;
}

.mobile.layout-preview {
    width: 108px;
    height: 192px;
    position: relative;
    overflow: hidden;
}
</style>
