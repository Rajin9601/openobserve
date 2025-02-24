<!-- Copyright 2023 Zinc Labs Inc.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU Affero General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
-->

<template>
  <div
    data-test="query-editor"
    class="logs-query-editor"
    ref="editorRef"
    id="editor"
  ></div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, watch } from "vue";
import * as monaco from "monaco-editor/esm/vs/editor/editor.api.js";
import "monaco-editor/esm/vs/language/css/monaco.contribution";

import { useStore } from "vuex";
import { getPath } from "@/utils/zincutils";

export default defineComponent({
  props: {
    query: {
      type: String,
      default: "",
    },
    fields: {
      type: Array,
      default: () => [],
    },
    functions: {
      type: Array,
      default: () => [],
    },
  },
  emits: ["update-query", "run-query"],
  setup(props, { emit }) {
    const store = useStore();
    const editorRef: any = ref();
    let editorObj: any = null;

    const createDependencyProposals = (range: any) => {
      const keywords = [
        {
          label: "and",
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: "and ",
          range: range,
        },
        {
          label: "or",
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: "or ",
          range: range,
        },
        {
          label: "like",
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: "like '%${1:params}%' ",
          range: range,
        },
        {
          label: "in",
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: "in ('${1:params}') ",
          range: range,
          insertTextRules:
            monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
        },
        {
          label: "not in",
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: "not in ('${1:params}') ",
          range: range,
          insertTextRules:
            monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
        },
        {
          label: "between",
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: "between '${1:params}' and '${1:params}' ",
          range: range,
          insertTextRules:
            monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
        },
        {
          label: "not between",
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: "not between '${1:params}' and '${1:params}' ",
          range: range,
          insertTextRules:
            monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
        },
        {
          label: "is null",
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: "is null ",
          range: range,
        },
        {
          label: "is not null",
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: "is not null ",
          range: range,
        },
        {
          label: ">",
          kind: monaco.languages.CompletionItemKind.Operator,
          insertText: "> ",
          range: range,
        },
        {
          label: "<",
          kind: monaco.languages.CompletionItemKind.Operator,
          insertText: "< ",
          range: range,
        },
        {
          label: ">=",
          kind: monaco.languages.CompletionItemKind.Operator,
          insertText: ">= ",
          range: range,
        },
        {
          label: "<=",
          kind: monaco.languages.CompletionItemKind.Operator,
          insertText: "<= ",
          range: range,
        },
        {
          label: "<>",
          kind: monaco.languages.CompletionItemKind.Operator,
          insertText: "<> ",
          range: range,
        },
        {
          label: "=",
          kind: monaco.languages.CompletionItemKind.Operator,
          insertText: "= ",
          range: range,
        },
        {
          label: "!=",
          kind: monaco.languages.CompletionItemKind.Operator,
          insertText: "!= ",
          range: range,
        },
        {
          label: "()",
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: "(${1:condition}) ",
          range: range,
          insertTextRules:
            monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
        },
      ];

      props.fields.forEach((field: any) => {
        if (field.name == store.state.zoConfig.timestamp_column) {
          return;
        }
        let itemObj = {
          label: field.name,
          kind: monaco.languages.CompletionItemKind.Text,
          insertText: field.name,
          range: range,
          insertTextRules:
            monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
        };
        keywords.push(itemObj);
      });

      props.functions.forEach((field: any) => {
        let itemObj = {
          label: field.name,
          kind: monaco.languages.CompletionItemKind.Text,
          insertText: field.name + field.args,
          range: range,
          insertTextRules:
            monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
        };
        keywords.push(itemObj);
      });

      return keywords;
    };

    onMounted(async () => {
      monaco.editor.defineTheme("myCustomTheme", {
        base: "vs", // can also be vs-dark or hc-black
        inherit: true, // can also be false to completely replace the builtin rules
        rules: [{ token: "comment", background: "FFFFFF" }],
        colors: {
          "editor.foreground": "#000000",
          "editor.background": "#fafafa",
          "editorCursor.foreground": "#000000",
          "editor.lineHighlightBackground": "#FFFFFF",
          "editorLineNumber.foreground": "#000000",
          "editor.border": "#000000",
        },
      });

      monaco.languages.registerCompletionItemProvider("sql", {
        provideCompletionItems: function (model, position) {
          // find out if we are completing a property in the 'dependencies' object.
          var textUntilPosition = model.getValueInRange({
            startLineNumber: 1,
            startColumn: 1,
            endLineNumber: position.lineNumber,
            endColumn: position.column,
          });

          var word = model.getWordUntilPosition(position);
          var range = {
            startLineNumber: position.lineNumber,
            endLineNumber: position.lineNumber,
            startColumn: word.startColumn,
            endColumn: word.endColumn,
          };

          let arr = textUntilPosition.trim().split(" ");
          let filteredSuggestions = [];
          filteredSuggestions = createDependencyProposals(range);
          filteredSuggestions = filteredSuggestions.filter((item) => {
            return item.label.toLowerCase().includes(word.word.toLowerCase());
          });

          // if (filteredSuggestions.length == 0) {
          const lastElement = arr.pop();

          filteredSuggestions.push({
            label: `match_all('${lastElement}')`,
            kind: monaco.languages.CompletionItemKind.Text,
            insertText: `match_all('${lastElement}')`,
            range: range,
          });
          filteredSuggestions.push({
            label: `match_all_ignore_case('${lastElement}')`,
            kind: monaco.languages.CompletionItemKind.Text,
            insertText: `match_all_ignore_case('${lastElement}')`,
            range: range,
          });
          filteredSuggestions.push({
            label: `match_all_indexed('${lastElement}')`,
            kind: monaco.languages.CompletionItemKind.Text,
            insertText: `match_all_indexed('${lastElement}')`,
            range: range,
          });
          filteredSuggestions.push({
            label: `match_all_indexed_ignore_case('${lastElement}')`,
            kind: monaco.languages.CompletionItemKind.Text,
            insertText: `match_all_indexed_ignore_case('${lastElement}')`,
            range: range,
          });
          return {
            suggestions: filteredSuggestions,
          };
          // } else {
          //   return {
          //     suggestions: filteredSuggestions,
          //   };
          // }
        },
      });

      editorObj = monaco.editor.create(editorRef.value, {
        value: props.query,
        language: "sql",
        theme: store.state.theme == "dark" ? "vs-dark" : "myCustomTheme",
        showFoldingControls: "never",
        wordWrap: "on",
        lineNumbers: "on",
        lineNumbersMinChars: 0,
        overviewRulerLanes: 0,
        fixedOverflowWidgets: false,
        overviewRulerBorder: false,
        lineDecorationsWidth: 3,
        hideCursorInOverviewRuler: true,
        renderLineHighlight: "none",
        glyphMargin: false,
        folding: false,
        scrollBeyondLastColumn: 0,
        scrollBeyondLastLine: true,
        scrollbar: { horizontal: "auto", vertical: "visible" },
        find: {
          addExtraSpaceOnTop: false,
          autoFindInSelection: "never",
          seedSearchStringFromSelection: "never",
        },
        minimap: { enabled: false },
      });

      editorObj.onDidChangeModelContent((e: any) => {
        emit("update-query", editorObj.getValue());
      });

      editorObj.createContextKey("ctrlenter", true);
      editorObj.addCommand(
        monaco.KeyMod.CtrlCmd | monaco.KeyCode.Enter,
        function () {
          setTimeout(() => {
            emit("run-query");
          }, 300);
        },
        "ctrlenter"
      );

      window.addEventListener("click", () => {
        editorObj.layout();
      });

      // TODO: make this global
      // watch(() => store.state.theme, () => {
      //   console.log('theme updated', store.state.theme);
      //   monaco.editor.setTheme(store.state.theme == 'dark' ? 'vs-dark' : 'myCustomTheme');
      // })

      // editorObj.onDidBlurEditorWidget(() => {
      //   // onBlur();
      // });

      //   //   editorObj.dispose();
    });

    const setValue = (value: string) => {
      if (editorObj?.setValue) editorObj.setValue(value);
    };

    const resetEditorLayout = () => {
      editorObj.layout();
    };

    const getCursorIndex = () => {
      const currentPosition = editorObj.getPosition();
      const cursorIndex = editorObj.getModel().getOffsetAt(currentPosition) - 1;
      return cursorIndex || null;
    };

    return {
      editorRef,
      editorObj,
      setValue,
      resetEditorLayout,
      getCursorIndex,
    };
  },
});
</script>

<style scoped>
#editor {
  width: 100%;
  height: 78%;
  border-radius: 5px;
}
</style>

<style lang="scss">
.logs-query-editor {
  .monaco-editor,
  .monaco-editor .monaco-editor {
    padding: 0px 0px 0px 0px !important;

    .editor-widget .suggest-widget {
      z-index: 9999;
      display: flex !important;
      visibility: visible !important;
    }
  }
}
</style>
