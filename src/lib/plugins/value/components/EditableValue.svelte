<svelte:options immutable={true} />

<script lang="ts">
  import type { JSONPath } from 'immutable-json-patch'
  import { compileJSONPointer } from 'immutable-json-patch'
  import { isObjectOrArray, stringConvert } from '$lib/utils/typeUtils.js'
  import { createValueSelection, getFocusPath } from '$lib/logic/selection.js'
  import { getValueClass } from '$lib/plugins/value/components/utils/getValueClass.js'
  import EditableDiv from '../../../components/controls/EditableDiv.svelte'
  import type {
    FindNextInside,
    JSONParser,
    OnFind,
    OnJSONSelect,
    OnPasteJson,
    OnPatch,
    ValueNormalization
  } from '$lib/types.js'
  import { UpdateSelectionAfterChange } from '$lib/types.js'
  import { isEqual } from 'lodash-es'
  import type { ComponentType } from 'svelte'

  export let path: JSONPath
  export let value: unknown
  export let parser: JSONParser
  export let normalization: ValueNormalization
  export let enforceString: boolean
  export let onPatch: OnPatch
  export let onPasteJson: OnPasteJson
  export let onSelect: OnJSONSelect
  export let onFind: OnFind
  export let focus: () => void
  export let findNextInside: FindNextInside
  export let editableComponent: ComponentType = EditableDiv

  function convert(value: string): unknown {
    return enforceString ? value : stringConvert(value, parser)
  }

  function handleChangeValue(newValue: string, updateSelection: UpdateSelectionAfterChange) {
    onPatch(
      [
        {
          op: 'replace',
          path: compileJSONPointer(path),
          value: convert(normalization.unescapeValue(newValue))
        }
      ],
      (patchedJson, patchedState) => {
        // Leave the selection as is when it is no longer the path that we were editing here
        // This happens for example when the user clicks or double-clicks on another value
        // whilst editing a value
        if (patchedState.selection && !isEqual(path, getFocusPath(patchedState.selection))) {
          return undefined
        }

        const selection =
          updateSelection === UpdateSelectionAfterChange.nextInside
            ? findNextInside(path)
            : createValueSelection(path, false)

        return {
          state: {
            ...patchedState,
            selection
          }
        }
      }
    )

    focus()
  }

  function handleCancelChange() {
    onSelect(createValueSelection(path, false))
    focus()
  }

  function handlePaste(pastedText: string): void {
    try {
      const pastedJson = parser.parse(pastedText)
      if (isObjectOrArray(pastedJson)) {
        onPasteJson({
          path,
          contents: pastedJson
        })
      }
    } catch (err) {
      // silently ignore: thee pasted text is no valid JSON object or array,
      // no need to do anything
    }
  }

  function handleOnValueClass(value: string): string {
    return getValueClass(convert(normalization.unescapeValue(value)), parser)
  }
</script>

<svelte:component
  this={editableComponent}
  value={normalization.escapeValue(value)}
  onChange={handleChangeValue}
  onCancel={handleCancelChange}
  onPaste={handlePaste}
  {onFind}
  onValueClass={handleOnValueClass}
/>
