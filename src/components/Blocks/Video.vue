<template>
  <figure>
    <template v-if="embedSrc">
      <div
        ref="element"
        tabindex="0"
        class="k-editor-video-block-container"
        @keydown.backspace="$emit('remove')"
        @keydown.delete="$emit('remove')"
        @keydown.enter.prevent="$emit('append')"
        @keydown.up="$emit('prev')"
        @keydown.down="$emit('next')"
      >
        <iframe :src="embedSrc" />
        <div class="k-editor-video-block-overlay" />
      </div>
      <figcaption>
        <k-editable
          :content="attrs.caption"
          :breaks="true"
          :placeholder="$t('editor.blocks.video.caption.placeholder') + '…'"
          @input="caption"
        />
      </figcaption>
      <k-dialog ref="settings" @submit="$refs.settings.close()" size="medium">
        <k-form :fields="fields" v-model="attrs" @submit="$refs.settings.close()" />
      </k-dialog>
    </template>
    <template v-else>
      <div class="k-editor-video-block-container">
        <input
          ref="element"
          class="k-editor-video-block-input"
          type="text"
          placeholder="Enter a Youtube or Vimeo URL …"
          v-model="attrs.src"
          @keydown.up="$emit('prev')"
          @keydown.down="$emit('next')"
          @keydown.enter.prevent="$emit('append')"
        >
      </div>
    </template>
  </figure>
</template>

<script>
export default {
  inheritAttrs: false,
  icon: "video",
  props: {
    attrs: {
      type: Object,
      default() {
        return {
          src: null
        }
      }
    }
  },
  computed: {
    embedSrc() {

      var url = this.attrs.src;

      if (!url) {
        return false;
      }

      var youtubePattern = /^.*(youtu.be\/|v\/|u\/\w\/|embed\/|watch\?v=|\&v=)([^#\&\?]*).*/;
      var youtubeMatch   = url.match(youtubePattern);

      if (youtubeMatch) {
        return "https://www.youtube.com/embed/" + youtubeMatch[2];
      }

      var vimeoPattern = /vimeo\.com\/([0-9]+)/;
      var vimeoMatch   = url.match(vimeoPattern);

      if (vimeoMatch) {
        return "https://player.vimeo.com/video/" + vimeoMatch[1];
      }

      return false;

    },
    fields() {
      return {
        src: {
          label: this.$t('editor.blocks.video.src.label'),
          type: "text",
          icon: "video",
          placeholder: this.$t('editor.blocks.video.src.placeholder')
        },
        css: {
          label: this.$t('editor.blocks.video.css.label'),
          type: "text",
          icon: "code",
        }
      };
    }
  },
  methods: {
    caption(html) {
      this.$emit("input", {
        attrs: {
          ...this.attrs,
          caption: html
        }
      });
    },
    focus() {
      this.$refs.element.focus();
    },
    menu() {

      if (this.embedSrc) {
        return [
          {
            icon: "open",
            label: this.$t("editor.blocks.video.open"),
            click: this.open
          },
          {
            icon: "video",
            label: this.$t("editor.blocks.video.settings"),
            click: this.$refs.settings.open
          },
        ];
      } else {
        return [];
      }

    },
    open() {
      window.open(this.attrs.src);
    },
    replace() {
      this.$emit("input", {
        attrs: { src: null }
      });

      this.$nextTick(() => {
        this.focus();
      });
    }
  }
};
</script>

<style lang="scss">
.k-editor-video-block {
  margin: 1.5rem 0;
}
.k-editor-video-block-container {
  position: relative;
  padding-bottom: 56.25%;
  background: #2d2e36;
}
.k-editor-video-block-container:focus-within {
  outline: 2px solid #b5d7fe;
  outline-offset: 2px;
}
.k-editor-video-block-overlay,
.k-editor-video-block iframe {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border: 0;
}
.k-editor-video-block-container:focus-within .k-editor-video-block-overlay {
  display: none;
}
.k-editor-video-block figcaption {
  display: block;
  line-height: 1.5em;
  font-size: .875rem;
  padding-top: .75rem;
  text-align: center;
}
.k-editor-video-block-input {
  position: absolute;
  font: inherit;
  background: none;
  border: 0;
  width: 100%;
  padding: .5rem;
  height: 100%;
  text-align: center;
  color: #fff;
}
.k-editor-video-block-input:focus {
  outline: 0;
}
</style>
