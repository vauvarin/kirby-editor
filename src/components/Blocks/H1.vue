<script>
export default {
  extends: "paragraph",
  icon: "title",
  placeholder: "Heading 1",
  breaks: true,
  marks: [
    'italic',
    // 'strikeThrough',
    'code',
    'link'
  ]
};
</script>

<style lang="scss">
.k-editor-h1-block {
  margin-bottom: .75rem;
}
.k-editor-h1-block .k-editable-placeholder,
.k-editor-h1-block .ProseMirror {
  font-size: 1.75rem;
  font-weight: 600;
  line-height: 1.5em;
}
.k-editor-h1-block .ProseMirror code,
.k-editor-h2-block .ProseMirror code,
.k-editor-h3-block .ProseMirror code {
  position: relative;
  font-size: .925em;
  display: inline-block;
  line-height: 1;
  padding: .15em .25em;
  background: rgba(#000, .1);
  border-radius: 3px;
  font-family: SFMono-Regular, Consolas, Liberation Mono, Menlo, Courier, monospace;
}
.k-editor-h1-block .k-editor-block-options {
  top: 8px;
}
</style>
