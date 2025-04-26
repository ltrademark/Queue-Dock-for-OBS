<script setup>
import { reactive, onMounted, ref, watch } from 'vue'
import OBSWebSocket from 'obs-websocket-js'

const obs = new OBSWebSocket()
const checklist = reactive([
  {
    label: 'News Story 1',
    checked: false
  },
  {
    label: 'News Story 2',
    checked: false
  },
  {
    label: 'News Story 3',
    checked: false
  },
])
const newItem = ref('')
const currentItemSym = ref('▶')

const obsUrl = import.meta.env.VITE_OBS_URL
const obsPassword = import.meta.env.VITE_OBS_PASSWORD
const textNode = import.meta.env.VITE_OBS_TEXT_SOURCE

const connectOBS = async () => {
  // Basic validation for connection details
  if (!obsUrl || !textNode) {
    console.warn('OBS URL or Text Source Name is missing. Please check your .env file (VITE_OBS_URL, VITE_OBS_TEXT_SOURCE).');
    return; // Stop connection attempt if essential info is missing
  }

  try {
    await obs.connect(obsUrl, obsPassword); // Password can be undefined/empty if not needed
    console.log('Successfully connected to OBS WebSocket.');
    await updateObsText(); // Update OBS text on initial connection
  } catch (err) {
    console.error('Failed to connect to OBS WebSocket:', err);
  }
}

 const updateObsText = async () => {
  if (!textNode) {
      console.warn('OBS Text Source Name (VITE_OBS_TEXT_SOURCE) is not defined. Cannot update OBS text.');
      return;
  }

  const visibleItems = checklist.filter(item => !item.checked);

  let obsText = ''; // Default text

  if (visibleItems.length > 0) {
    const formattedItems = visibleItems.map((item, index) => {
      if (index === 0) {
        return `${currentItemSym.value} ${item.label}`; // Prepend symbol
      }
      // Using an ideographic space (U+3000) for consistent indentation
      return `　${item.label}`; // Indent other items
    });
    // Join the formatted items with newlines
    obsText = formattedItems.join('\n');
  } else if (checklist.length > 0) {
      // Optional: Message when list has items but all are checked
      obsText = '✅ All items complete!';
  } // If checklist is empty, obsText remains '' (or set a specific message)


  try {
    // Use the calculated 'obsText' variable
    await obs.call('SetInputSettings', {
      inputName: textNode, // Make sure this matches your OBS text source name exactly
      inputSettings: { text: obsText } // Use the calculated obsText
    });
    console.log('OBS text source updated successfully.');
  } catch (err) {
    console.error('Failed to update OBS text source:', err);
  }
}

function addItem() {
  const trimmed = newItem.value.trim(); // Access .value for ref
  if (trimmed) {
    // Use unshift to add the new item to the beginning of the array
    checklist.unshift({
      label: trimmed,
      checked: false
    });
    newItem.value = ''; // Clear the input (access .value for ref)
    // Watcher will automatically call updateObsText
  }
}

function removeItem(index) {
  // Splice directly from the reactive array (no .value needed)
  if (index >= 0 && index < checklist.length) {
    checklist.splice(index, 1);
    // Watcher will automatically call updateObsText
  } else {
    console.warn(`Attempted to remove item at invalid index: ${index}`);
  }
}

/**
 * Updates the label of a checklist item when its contenteditable span loses focus or Enter is pressed.
 * @param {number} index - The index of the item in the checklist array.
 * @param {Event} event - The blur or keydown event object.
 */
function updateLabel(index, event) {
  // Check if the item exists at the index
  if (checklist[index]) {
    const newLabel = event.target.textContent.trim();
    // Update only if the label has actually changed and is not empty
    if (newLabel && checklist[index].label !== newLabel) {
      console.log(`Updating label for index ${index} from "${checklist[index].label}" to "${newLabel}"`);
      checklist[index].label = newLabel;
      // The deep watcher on 'checklist' should automatically trigger updateObsText
    } else {
      // If the new label is empty or unchanged, revert the display to the original label
      // This prevents accidentally deleting labels or saving unchanged content
      event.target.textContent = checklist[index].label;
    }
  }
}

/**
 * Handles the Enter key press on the contenteditable span.
 * Prevents adding a new line and triggers blur to save the change.
 * @param {KeyboardEvent} event - The keydown event object.
 */
function handleLabelEnter(event) {
  event.preventDefault(); // Prevent adding a new line
  event.target.blur(); // Trigger the blur event to save the change
}

onMounted(() => {
  connectOBS(); // Attempt to connect when the component mounts
});

// --- Watchers ---

// Watch the checklist for changes (deep: true is needed for changes within objects in the array, including label edits)
watch(checklist, async (newChecklist, oldChecklist) => {
  console.log('Checklist changed, updating OBS text...');
  // Log the change for debugging label edits
  // console.log('Old:', JSON.stringify(oldChecklist));
  // console.log('New:', JSON.stringify(newChecklist));
  await updateObsText(); // Update OBS whenever the list changes
}, { deep: true }); // Deep watcher is crucial here

// Watch for changes in the symbol input
watch(currentItemSym, async (newSymbol, oldSymbol) => {
    if (newSymbol !== oldSymbol) {
        console.log('Symbol changed, updating OBS text...');
        await updateObsText();
    }
});

</script>

<template>
  <div class="queue-body">
    <div class="queue-body--header">
      <h2>Queue</h2>
      <input class="symbol-input" type="text" maxlength="1" v-model="currentItemSym" title="Symbol to denote current/top Item">
    </div>
    <div class="queue-body--controls">
      <input v-model="newItem" @keydown.enter="addItem" placeholder="Add a new Item" aria-label="New Item" />
      <button @click="addItem" aria-label="Add Item">
        <svg class="queue-icon" viewBox="0 0 24 24" width="24" height="24" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round"><line x1="12" y1="5" x2="12" y2="19"></line><line x1="5" y1="12" x2="19" y2="12"></line></svg>
      </button>
    </div>
    <transition-group name="list" tag="ul" class="queue-body--list">
       <li class="queue-body--list_item" 
           v-for="(item, index) in checklist" 
           :key="item.label + '-' + index"
           :style="{'opacity': item.checked ? '.54' : '1'}"> 
        <input type="checkbox"
                v-model="item.checked"
                :aria-labelledby="'task-label-' + index"/>
        <label>
          <span :id="'task-label-' + index"
                :class="{ 'item-checked': item.checked }"
                :contenteditable="!item.checked" 
                @blur="updateLabel(index, $event)" 
                @keydown.enter="handleLabelEnter($event)">
            {{ item.label }}
          </span>
        </label>
        <button @click="removeItem(index)" aria-label="Remove" class="remove-button">
          <svg class="queue-icon" viewBox="0 0 24 24" width="24" height="24" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="6" x2="6" y2="18"></line><line x1="6" y1="6" x2="18" y2="18"></line></svg>
        </button>
      </li>
    </transition-group>
    <p v-if="checklist.length === 0" class="empty-message">No Items yet. Add some!</p>
  </div>
</template>

<style lang="scss">
  :root {
    --offset-val: 90%;
    --contrast: 49.44;
		--ratio: calc((var(--contrast) - l) * infinity);
    // Color Palette
    --accent: #15e;
    --accent-offset: color-mix(in srgb, var(--accent) var(--offset-val), #000);
    --foreground: #fff;
    --foreground-offset: color-mix(in srgb, var(--foreground) var(--offset-val), #000);
    --background: #131313;
    --background-offset: color-mix(in srgb, var(--background) var(--offset-val), #fff);

    --border-color: color-mix(in srgb, var(--foreground) 30%, #000);

    --gutter: 16px;
    --padding: var(--gutter);
    --br: 8px;
    --br-inner: calc(var(--br) - var(--padding));
    --br-outer: calc(var(--br) + var(--padding));

    // Typography
    --font-family-sans: monospace;
    --font-size-base: 1rem; // 16px default
    --font-size-lg: 1.25rem;
    --font-size-xl: 2rem;
    --font-weight-normal: 400;
    --font-weight-medium: 500;
    --font-weight-semibold: 600;
    --font-weight-bold: 900;

    // Transitions
    --transition-fast: 0.2s ease;
  }

  *,
  *:before,
  *:after {
    box-sizing: border-box;
  }

  html, body, #app {
    font-family: var(--font-family-sans);
    margin: 0;
    padding: 0;
    height: 100%;
    width: 100%;
    color: var(--foreground);
    background-color: var(--background);
  }

  body {
    display: block;
    overflow: hidden;
    accent-color: var(--color-primary);
  }

  #app {
    overflow: auto;
    display: flex;
  }

  .queue-body {
    display: flex;
    flex-flow: column;
    overflow-x: hidden;
    width: 100%;

    &--header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      font-weight: var(--font-weight-semibold);
      font-size: var(--font-size-xl);
      margin: 0;
      padding: var(--padding);
      border-bottom: 1px solid var(--border-color);
      :is(h1, h2, h3) {
        margin: 0;
        padding: 0;
        font-size: inherit;
        font-weight: inherit;
      }
      .symbol-input {
        padding:0;
        margin: 0;
        color: inherit;
        background: none;
        text-align: center;
        border: none;
        outline: none;
        width: 50px;
        aspect-ratio: 1;
        border-radius: 25px;
        background-color: var(--background-offset);
        &:focus {
          outline: 1px solid var(--accent);
          background-color: color-mix(in srgb, var(--accent) 20%, transparent);
        }
      }
    }

    &--controls {
      display: flex;
      border-bottom: 1px solid var(--border-color);

      input {
        flex-grow: 1;
        padding: calc(var(--padding) / 2) var(--padding);
        background-color: var(--background-offset);
        color: inherit;
        border: none;
        outline: 1px solid var(--border-color);
        font-size: var(--font-size-base);
        transition: border-color var(--transition-fast), box-shadow var(--transition-fast);

        &:focus {
          outline: 1px solid var(--accent);
          background-color: color-mix(in srgb, var(--accent) 20%, transparent);
        }
      }

      button {
        cursor: pointer;
        display: grid;
        align-content: center;
        aspect-ratio: 1;
        padding: var(--padding);
        background-color: var(--accent);
        color: lch(from var(--accent) var(--ratio) 0 0);
        border: none;
        font-size: 22px;
        font-weight: var(--font-weight-bold);
        transition: background-color var(--transition-fast);
        flex-shrink: 0;

        &:hover {
          background-color: var(--accent-offset);
        }
      }
    }

    &--list {
      display: flex;
      flex-direction: column;
      gap: var(--padding);
      list-style-type: none;
      margin: 0;
      padding: var(--padding);

      &_item {
        --inner-pads: calc(var(--padding) / 2);
        display: flex;
        align-items: center;
        justify-content: space-between;
        padding: calc(var(--inner-pads) / 2) var(--inner-pads);
        gap: var(--inner-pads);
        background-color: var(--background-offset);
        border-radius: var(--br);
        border: 1px solid var(--border-color);
        transition: background-color var(--transition-fast);
        
        &:hover {
          background-color: color-mix(in srgb, var(--accent) 20%, transparent);
        }
        
        input[type="checkbox"] {
          cursor: pointer;
          width: 1.5em;
          aspect-ratio: 1;
          flex-shrink: 0;
          border-radius: var(--br);
          background-color: var(--background-offset);
          margin: 0;
        }

        label {
          display: flex;
          align-items: center;
          gap: var(--padding);
          flex-grow: 1;
          width: calc(100% - ((var(--padding) * 3) + 16px));

          span {
            cursor: default;
            display: block;
            width: 100%;
            text-overflow: ellipsis;
            white-space: nowrap;
            overflow: hidden;
            color: inherit;
            transition: color var(--transition-fast), text-decoration var(--transition-fast);
            flex-grow: 1;
            &[contenteditable="true"] {
              cursor: text;
            }

            &:focus {
              outline: none;
              border-bottom:1px dashed currentColor;
            }

            &.item-checked {
              color: var(--accent);
              text-decoration: line-through;
              font-style: italic;
            }
          }
        }

        .remove-button {
          --color-danger: #f00;
          cursor: pointer;
          display: grid;
          align-content: center;
          aspect-ratio: 1;
          background: none;
          border: none;
          color: var(--color-danger);
          font-size: 18px;
          line-height: 1;
          border-radius: var(--br-inner);
          transition: background-color var(--transition-fast), color var(--transition-fast);
          flex-shrink: 0;
          padding: calc(var(--padding) / 2);
          margin-right: calc((var(--padding) / 4) * -1);

          &:hover {
            background-color: color-mix(in srgb, var(--color-danger) 20%, transparent);
          }
        }
      }
    }

    .empty-message {
      text-align: center;
      color: var(--foreground-offset);
      padding: var(--padding);
      font-style: italic;
    }
  }

  .queue-icon {
    display: inline-block;
    width: auto;
    height: 1em;
    margin: 0;
    padding: 0;
  }

  /* --- List Transition Animations --- */
  .list-enter-active {
    transition: all var(--transition-medium) ease-out;
  }
  .list-enter-from {
    opacity: 0;
    transform: translateY(-15px);
  }

  .list-leave-active {
    transition: all var(--transition-medium) ease-in;
    position: absolute;
    width: 100%;
  }
  .list-leave-to {
    opacity: 0;
    transform: translateY(-15px);
  }
  .list-move {
    transition: transform var(--transition-medium) ease;
  }
</style>