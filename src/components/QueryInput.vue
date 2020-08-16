<template>
  <div>
    <div :class="$style.placeholder">{{ placeholder }}</div>
    <div
      contenteditable="true"
      role="textarea"
      :class="[dataClassName, $style.editor]"
      @beforeinput="onBeforeInput"
      @input="onInput"
      @keydown="onKeyDown"
    >
      <span v-for="(node, i) in inputNodes" :key="i">
        <span :data-index="i" v-if="node.type === 'text'">{{
          node.value
        }}</span>
        <span
          :data-index="i"
          v-if="node.type === 'query'"
          :class="$style.query"
          >{{ node.value }}</span
        >
      </span>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, nextTick, toRaw } from "vue";

type InputNode = {
  /** value for node. */
  value: string;
} & (
  | { type: "text" }
  | {
      type: "query";
      queryType: string;
    }
);

const deepCopy = (a: object) => JSON.parse(JSON.stringify(a));

const generateRandomString = () =>
  Math.random()
    .toString(32)
    .substring(2);

const useQueryPrettiier = (queryRegExp?: RegExp) => {
  const dataClassName = `query-input-${generateRandomString()}`;
  const undoHistory: InputNode[][] = [];
  const redoHistory: InputNode[][] = [];

  const inputNodes = ref<InputNode[]>([{ type: "text", value: "" }]);

  const makeAdjustCursor = (targetIndex: number, targetOffset: number) => (
    adjust = 0
  ) => {
    nextTick(() => {
      // adjust selection
      const node =
        document.querySelector(
          `.${dataClassName} [data-index='${targetIndex}']`
        )?.firstChild ?? null;
      getSelection()?.collapse(node, targetOffset + adjust);
    });
  };

  const onBeforeInput = (event: InputEvent) => {
    console.log(event.inputType);

    event.preventDefault();

    const selection = getSelection();
    if (!selection) {
      return;
    }

    const anchorOffset = selection.anchorOffset;
    const focusOffset = selection.focusOffset;
    const insertLength = event.data?.length ?? 0;
    const anchorElementIndex = parseInt(
      selection?.anchorNode?.parentElement?.dataset["index"] ?? "0"
    );
    const focusElementIndex = parseInt(
      selection?.focusNode?.parentElement?.dataset["index"] ?? "0"
    );

    let targetIndex = anchorElementIndex;

    // inputNodes index
    const leadingIndex = Math.min(anchorElementIndex, focusElementIndex);
    const trailingIndex = Math.max(anchorElementIndex, focusElementIndex);

    // selection offset
    const leadingOffset =
      leadingIndex === trailingIndex
        ? Math.min(anchorOffset, focusOffset)
        : anchorElementIndex === leadingIndex
        ? anchorOffset
        : focusOffset;
    const trailingOffset =
      leadingIndex === trailingIndex
        ? Math.max(anchorOffset, focusOffset)
        : anchorElementIndex === trailingIndex
        ? anchorOffset
        : focusOffset;

    // record history and clear forward history
    undoHistory.push(deepCopy(toRaw(inputNodes.value)));
    redoHistory.splice(0, redoHistory.length);

    // update value
    inputNodes.value[leadingIndex].value =
      inputNodes.value[leadingIndex].value.substring(0, leadingOffset) +
      inputNodes.value[trailingIndex].value.substring(trailingOffset);

    // collapse array if trailing and leading index are different
    inputNodes.value.splice(leadingIndex + 1, trailingIndex - leadingIndex);
    targetIndex = leadingIndex;

    const targetNode = inputNodes.value[targetIndex];

    const adjustCursor = makeAdjustCursor(targetIndex, leadingOffset);

    if (
      event.inputType === "insertText" ||
      event.inputType === "insertCompositionText"
    ) {
      targetNode.value =
        targetNode.value.substring(0, anchorOffset) +
        event.data +
        targetNode.value.substring(anchorOffset);
      adjustCursor(insertLength);
    }
    if (
      event.inputType === "insertFromPaste" ||
      event.inputType === "insertReplacementText"
    ) {
      const content =
        ((event as any)?.dataTransfer as DataTransfer | undefined)?.getData(
          "text"
        ) ?? "";
      targetNode.value =
        targetNode.value.substring(0, anchorOffset) +
        content +
        targetNode.value.substring(anchorOffset);
      adjustCursor(content.length);
    }
    if (event.inputType === "deleteContentForward") {
      if (leadingIndex === trailingIndex && leadingOffset === trailingOffset) {
        // selected case are already done
        targetNode.value =
          targetNode.value.substring(0, leadingOffset) +
          targetNode.value.substring(leadingOffset + 1);
      }
      adjustCursor();
    }

    if (event.inputType === "deleteContentBackward") {
      if (leadingIndex === trailingIndex && leadingOffset === trailingOffset) {
        // selected case are already done
        targetNode.value =
          targetNode.value.substring(0, leadingOffset - 1) +
          targetNode.value.substring(leadingOffset);
        adjustCursor(-1);
      } else {
        adjustCursor();
      }
    }
  };

  const onInput = (event: InputEvent) => {
    console.log(event.inputType);
  };
  const onKeyDown = (event: KeyboardEvent) => {
    const selection = getSelection();
    const anchorOffset = selection?.anchorOffset ?? 0;
    const anchorElementIndex = parseInt(
      selection?.anchorNode?.parentElement?.dataset["index"] ?? "0"
    );

    const adjustCursor = makeAdjustCursor(anchorElementIndex, anchorOffset - 1);

    if (event.metaKey && !event.shiftKey && event.key === "z") {
      // undo
      const undoEntry = undoHistory.pop();
      if (undoEntry) {
        redoHistory.push(deepCopy(toRaw(inputNodes.value)));
        inputNodes.value = [...undoEntry];
        adjustCursor();
      }
    }
    if (event.metaKey && event.shiftKey && event.key === "z") {
      // redo
      const redoEntry = redoHistory.pop();
      if (redoEntry) {
        undoHistory.push(deepCopy(toRaw(inputNodes.value)));
        inputNodes.value = [...redoEntry];
        adjustCursor();
      }
    }
  };
  return { dataClassName, onBeforeInput, onInput, onKeyDown, inputNodes };
};

export default defineComponent({
  name: "QueryInput",
  props: {
    placeholder: {
      type: String,
      default: ""
    }
  },
  setup() {
    const {
      onInput,
      onBeforeInput,
      onKeyDown,
      inputNodes,
      dataClassName
    } = useQueryPrettiier();
    return { onInput, onBeforeInput, onKeyDown, inputNodes, dataClassName };
  }
});
</script>

<style lang="scss" module>
.editor {
  padding: 0.5rem;
  white-space: pre-wrap;
}
.query {
  background: #f0f2f5;
  color: #6b7d8a;
  font-weight: bold;
  margin: 0 0.5rem;
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
}
.placeholder {
  display: none;
}
</style>
