<template>
  <div
    ref="editor"
    :spellcheck="spellcheck"
    class="k-editable"
    @focusin="onFocus"
    @focusout="onBlur"
  >
    <k-editable-toolbar
      v-if="editor"
      v-show="toolbar"
      ref="toolbar"
      :marks="toolbar.marks"
      :options="editor.state.schema.marks"
      :style="{bottom: toolbar.bottom + 'px', left: toolbar.left + 'px'}"
      @option="onOption"
    />
    <span class="k-editable-placeholder" v-if="placeholder && editor && isEmpty()">{{ placeholder }}</span>
    <k-link-dialog ref="link" @submit="insertLink" />
  </div>
</template>

<script>
/** ProseMirror */
import { TextSelection, AllSelection } from "prosemirror-state";
import { DOMSerializer, Slice, Fragment } from "prosemirror-model";
import Doc from "./Editor/Document.js";

/** Editor wrapper */
import Editor from "./Editor/View.js";

/** Commands */
import { toggleMark } from "prosemirror-commands";

/** Utils */
import {
  getActiveMarks,
  getMarkAttrs,
  getHTML,
} from "./Utils.js";

/** Dialogs */
import LinkDialog from "./Dialogs/Link.vue";

/** Toolbar */
import Toolbar from "./Toolbar.vue";

export default {
  inheritAttrs: false,
  components: {
    "k-link-dialog": LinkDialog,
    "k-editable-toolbar": Toolbar
  },
  fields: {
    link: {
      href: {
        label: "URL",
        type: "text",
        icon: "url"
      }
    }
  },
  props: {
    content: {
      type: String,
      default: ""
    },
    breaks: Boolean,
    code: Boolean,
    spellcheck: Boolean,
    paste: {
      type: Function,
      default() {
        return function () {};
      }
    },
    placeholder: String,
    marks: {
      type: Array,
      default() {
        return [
          "bold",
          "code",
          "italic",
          "link",
          // "strikeThrough",
          // "underline",
        ];
      }
    }
  },
  data() {
    return {
      editor: null,
      toolbar: false
    };
  },
  mounted() {
    this.editor = Editor({
      breaks: this.breaks,
      code: this.code,
      content: this.content,
      element: this.$el,
      marks: this.marks,
      onUpdate: this.onUpdate,
      onSplit: this.onSplit,
      onBack: this.onBack,
      onBold: this.onBold,
      onConvert: this.onConvert,
      onItalic: this.onItalic,
      onLink: this.onLink,
      onNext: this.onNext,
      onPaste: this.onPaste,
      onPrev: this.onPrev,
      onEnter: this.onEnter,
      onSelect: this.onSelect,
      onShiftEnter: this.onShiftEnter,
      onShiftTab: this.onShiftTab,
      onStrikeThrough: this.onStrikeThrough,
      onTab: this.onTab,
      onUnderline: this.onUnderline,
    });

    this.onUpdate();
  },
  destroyed() {
    this.editor.destroy();
  },
  methods: {
    addMark(type, attrs) {
      const { from, to } = this.selection();
      const mark = this.mark(type);

      if (mark) {
        return this.dispatch(this.tr().addMark(from, to, mark.create(attrs)))
      }
    },
    coordsAtPos(pos) {
      return this.editor.coordsAtPos(pos);
    },
    coordsAtEnd() {
      return this.editor.coordsAtPos(this.cursorPositionAtEnd());
    },
    coordsAtStart() {
      return this.editor.coordsAtPos(0);
    },
    coordsAtCursor() {
      return this.coordsAtPos(this.cursorPosition());
    },
    posAtCoords(coords) {
      return this.editor.posAtCoords(coords);
    },
    cursor() {
      let { $cursor } = this.selection();
      return $cursor;
    },
    cursorAtEnd() {
      let { $cursor } = this.selectionAtEnd();
      return $cursor;
    },
    cursorAtStart() {
      let { $cursor } = this.selectionAtStart();
      return $cursor;
    },
    cursorPosition() {
      const $cursor = this.cursor();
      return $cursor ? $cursor.pos : 0;
    },
    cursorPositionAtEnd() {
      const $cursor = this.cursorAtEnd();
      return $cursor ? $cursor.pos : 0;
    },
    cursorPositionAtStart() {
      return 0;
    },
    dispatch(tr) {
      this.editor.dispatch(tr);
    },
    doc() {
      return this.editor.state.doc;
    },
    focus(cursor) {

      if (cursor) {

        if (typeof cursor === "object") {

          const at     = cursor.at || "start";
          const coords = (at === "end") ? this.coordsAtEnd() : this.coordsAtStart();

          const pos = this.posAtCoords({
            top: coords.top,
            left: cursor.left || 0
          });

          if (pos && pos.pos) {
            cursor = pos.pos;
          } else {
            cursor = at;
          }

        }

        let selection = null;

        switch (cursor) {
          case "start":
            selection = TextSelection.atStart(this.doc());
            break;
          case "end":
            selection = TextSelection.atEnd(this.doc());
            break;
          default:
            try {
              selection = TextSelection.near(this.doc().resolve(cursor));
            } catch (e) {
              selection = TextSelection.atStart(this.doc());
            }
            break;
        }

        this.dispatch(this.tr().setSelection(selection));

      }

      setTimeout(() => {
        this.editor.focus();
      }, 1);

    },
    getActiveMarks() {
      return getActiveMarks(this.editor.state.schema, this.editor.state, this.marks);
    },
    getMarkAttrs(type) {
      return getMarkAttrs(this.state(), type);
    },
    hasFocus() {
      return this.editor.hasFocus;
    },
    hasMark(type) {
      return this.editor.state.schema.marks[type] !== undefined;
    },
    htmlAtSelection(selection) {
      return this.nodeToHtml(this.nodeAtSelection(selection));
    },
    htmlBeforeCursor() {
      return this.htmlAtSelection(this.selectionBeforeCursor());
    },
    htmlAfterCursor() {
      return this.htmlAtSelection(this.selectionAfterCursor());
    },
    insertBreak() {
      if (this.breaks !== true) {
        return false;
      }

      if (this.code) {
        this.dispatch(
            this.tr()
              .insertText("\n")
              .scrollIntoView()
        );
      } else {
        this.dispatch(
          this.tr()
            .replaceSelectionWith(this.schema().nodes.hard_break.create())
            .scrollIntoView()
        );
      }

    },
    insertHtml(html) {
      const node = Doc(this.schema(), html);
      this.dispatch(this.tr().replaceSelectionWith(node).scrollIntoView());
    },
    insertLink(href) {
      if (!href) {
        this.removeMark("link");
      } else {
        this.addMark("link", {
          href: href
        });
      }
    },
    insertText(text) {
      this.dispatch(this.tr().insertText(text).scrollIntoView());
    },
    isEmpty() {
      return this.doc().content.size === 0;
    },
    isSelected() {
      const selection = this.selection();
      const end       = this.cursorPositionAtEnd();

      return selection.from === 0 && selection.to === end;
    },
    link() {
      const attrs = this.getMarkAttrs("link");
      this.$refs.link.open(attrs.href);
    },
    mark(type) {
      return this.editor.state.schema.marks[type];
    },
    nodeAtSelection(selection) {
      return this.doc().cut(selection.from, selection.to);
    },
    nodeToHtml(node) {
      const result = DOMSerializer
        .fromSchema(this.editor.state.schema)
        .serializeFragment(node);

      let dom = document.createElement("div");
      dom.append(result);

      return dom.innerHTML;
    },
    onBack() {
      this.$emit("back", {
        html: this.htmlAfterCursor()
      });
    },
    onBold() {
      this.toggleMark("bold");
    },
    onBlur() {
      this.toolbar = false;
      this.$emit("blur");
    },
    onConvert(type) {
      this.$emit("convert", type);
    },
    onEnter() {
      if (this.code) {
        this.insertBreak();
      }

      this.$emit("enter", event);
    },
    onFocus() {
      this.$emit("focus", event);
    },
    onInput(html) {
      this.$emit("input", html);
    },
    onItalic() {
      this.toggleMark("italic");
    },
    onLink() {
      this.link();
    },
    onNext() {
      let { left } = this.coordsAtCursor();
      this.$emit("next", {
        left: left,
        at: "start",
      });
    },
    onOption(option) {
      if (!this[option.action]) {
        return false;
      }

      const args = option.args || [];

      this[option.action](...args);
    },
    onPaste(html, text) {
      this.$emit("paste", { html, text });
    },
    onPrev() {
      let { left } = this.coordsAtCursor();

      this.$emit("prev", {
        left: left,
        at: "end"
      });
    },
    onSelect() {

      const selection = this.editor.state.selection;

      if (selection.empty) {
        this.toolbar = false;
        this.$emit("deselect");
      } else {

        const toolbar = this.$refs.toolbar;

        if (!toolbar) {
          return false;
        }

        const { from, to } = selection;

        const start = this.coordsAtPos(from);
        const end   = this.coordsAtPos(to, true);

        // The box in which the tooltip is positioned, to use as base
        const box = this.$el.getBoundingClientRect();
        const el  = toolbar.$el.getBoundingClientRect();

        // Find a center-ish x position from the selection endpoints (when
        // crossing lines, end may be more to the left)
        let left   = ((start.left + end.left) / 2) - box.left
        let bottom = Math.round(box.bottom - start.top);

        this.toolbar = {
          left: left,
          bottom: bottom,
          marks: this.getActiveMarks()
        };

        this.$emit("select");
      }
    },
    onShiftEnter() {
      this.$emit("shiftEnter");
      this.insertBreak();
    },
    onShiftTab() {
      this.$emit("shiftTab");
    },
    onStrikeThrough() {
      this.toggleMark("strikeThrough");
    },
    onSplit() {
      this.$emit("split", {
        cursor: this.cursorPosition(),
        before: this.htmlBeforeCursor(),
        after: this.htmlAfterCursor()
      });
    },
    onTab() {
      if (this.code) {
        this.insertText("\t");
      }

      this.$emit("tab");
    },
    onUnderline() {
      this.toggleMark("underline");
    },
    onUpdate() {
      this.onInput(this.toHTML());
    },
    removeMark(type) {
      const { from, to } = this.selection();
      const mark = this.mark(type);

      if (mark) {
        return this.dispatch(this.tr().removeMark(from, to, mark));
      }
    },
    schema() {
      return this.editor.state.schema;
    },
    selection() {
      return this.editor.state.selection;
    },
    selectionAtEnd() {
      return TextSelection.atEnd(this.doc());
    },
    selectionAtStart() {
      return TextSelection.atStart(this.doc());
    },
    selectionBeforeCursor() {
      return new TextSelection(this.doc().resolve(0), this.selection().$from);
    },
    selectionAfterCursor() {
      return new TextSelection(this.selection().$to, this.selectionAtEnd().$to);
    },
    state() {
      return this.editor.state;
    },
    toggleMark(type, attrs) {
      const mark = this.mark(type);

      if (mark) {
        toggleMark(mark, attrs)(this.editor.state, this.editor.dispatch);
      }

      this.editor.focus();
    },
    toHTML() {
      return getHTML(this.editor.state, this.code);
    },
    toJSON() {
      return this.doc().toJSON();
    },
    tr() {
      return this.editor.state.tr;
    },
    view() {
      return this.editor;
    }
  }
};
</script>

<style>
.k-editable {
  position: relative;
  width: 100%;
}
.k-editable .ProseMirror {
  word-wrap: break-word;
  white-space: pre-wrap;
  -webkit-font-variant-ligatures: none;
  font-variant-ligatures: none;
}
.k-editable-placeholder {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  color: #bbb;
  pointer-events: none;
}
</style>
