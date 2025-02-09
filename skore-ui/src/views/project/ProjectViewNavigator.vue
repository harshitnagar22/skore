<script setup lang="ts">
import { autoUpdate, useFloating } from "@floating-ui/vue";
import Simplebar from "simplebar-vue";
import { onBeforeUnmount, onMounted, ref } from "vue";

import EditableList, { type EditableListItemModel } from "@/components/EditableList.vue";
import { generateRandomId } from "@/services/utils";
import { useProjectStore } from "@/stores/project";
import { useToastsStore } from "@/stores/toasts";

const projectStore = useProjectStore();
const toastsStore = useToastsStore();
const isDropdownOpen = ref(false);
const el = ref<HTMLElement>();
const dropdown = ref<HTMLDivElement>();
const floating = ref<HTMLDivElement>();
const { floatingStyles } = useFloating(dropdown, floating, {
  placement: "bottom",
  strategy: "fixed",
  whileElementsMounted: autoUpdate,
});
const views = ref<EditableListItemModel[]>([]);
let unsavedViewsId: string = "";

function onViewSelected(view: string) {
  projectStore.setCurrentView(view);
  isDropdownOpen.value = false;
  populateViewList();
}

async function onViewRenamed(oldName: string, newName: string, item: EditableListItemModel) {
  const viewAlreadyExists = views.value.filter((view) => view.name === newName).length > 1;
  if (viewAlreadyExists) {
    toastsStore.addToast(`A view named "${newName}" already exists`, "error");
    item.isNamed = false;
    return;
  }
  // user can rename an existing view
  // and rename a new view that is not known by the backend
  if (unsavedViewsId == item.id) {
    await projectStore.createView(newName);
    unsavedViewsId = "";
    toastsStore.addToast("View added successfully", "success", { duration: 20 });
  } else {
    await projectStore.renameView(oldName, newName);
  }
  // auto switch the view to the renamed one
  onViewSelected(newName);
}

async function onViewsListAction(action: string, item: EditableListItemModel) {
  switch (action) {
    case "rename": {
      item.isNamed = false;
      break;
    }
    case "duplicate": {
      const index = views.value.indexOf(item) ?? 0;
      const newName = `${item.name} - copy`;
      const id = generateRandomId();
      views.value.splice(index + 1, 0, {
        name: newName,
        isNamed: false,
        id,
      });
      unsavedViewsId = id;
      await projectStore.duplicateView(item.name, newName);
      break;
    }
    case "delete": {
      views.value.splice(views.value.indexOf(item), 1);
      if (item.isNamed) {
        await projectStore.deleteView(item.name);
        toastsStore.addToast("View deleted successfully", "success", { duration: 20 });
      }
      break;
    }
  }
}

function onAddView() {
  const hasUnsavedViews = views.value.some((view) => !view.isNamed);
  if (!hasUnsavedViews) {
    const search = /New view ?(\d+)?/;
    const viewsNamedWithDefaultCount = views.value.reduce((acc, view) => {
      const match = view.name.match(search);
      if (match && match.length > 1) {
        return parseInt(match[1] ?? 0) + 1;
      }
      return acc;
    }, 0);
    const id = generateRandomId();
    const suffix = viewsNamedWithDefaultCount > 0 ? ` ${viewsNamedWithDefaultCount}` : "";
    views.value.push({ name: `New view${suffix}`, isNamed: false, id });
    unsavedViewsId = id;
  }
}

function onClickOutside(e: Event) {
  if (el.value) {
    // is it a click outside and do we have a pending action ?
    const isOutside = !el.value.contains(e.target as Node);
    const isPendingRename = views.value.some((view) => !view.isNamed) || unsavedViewsId !== "";
    if (isOutside && !isPendingRename) {
      isDropdownOpen.value = false;
    }
  }
}

function populateViewList() {
  views.value = Object.keys(projectStore.views).map((key) => ({
    name: key,
    isNamed: true,
    id: generateRandomId(),
    isSelected: key === projectStore.currentView,
  }));
}

onMounted(async () => {
  populateViewList();
  document.addEventListener("click", onClickOutside);
});

onBeforeUnmount(() => {
  document.removeEventListener("click", onClickOutside);
});
</script>

<template>
  <div class="project-view-navigator" ref="el">
    <div class="dropdown" @click="isDropdownOpen = !isDropdownOpen" ref="dropdown">
      <i class="icon-recent-document" />
      <span class="current-view-name">{{ projectStore.currentView ?? "Select a view" }}</span>
      <i class="icon-chevron-up" :class="{ turned: isDropdownOpen }" />
    </div>
    <Transition name="fade">
      <div class="dropdown-menu" v-if="isDropdownOpen" ref="floating" :style="floatingStyles">
        <div class="new-view" @click="onAddView">
          <span>Add a new view</span>
          <i class="icon-plus-circle" />
        </div>
        <Simplebar class="view-list simplebar-inverted" v-if="views.length > 0">
          <EditableList
            v-model:items="views"
            :actions="[
              { label: 'rename', emitPayload: 'rename', icon: 'icon-edit' },
              { label: 'duplicate', emitPayload: 'duplicate', icon: 'icon-copy' },
              { label: 'delete', emitPayload: 'delete', icon: 'icon-trash' },
            ]"
            @action="onViewsListAction"
            @select="onViewSelected"
            @rename="onViewRenamed"
          />
        </Simplebar>
      </div>
    </Transition>
  </div>
</template>

<style scoped>
.project-view-navigator {
  display: flex;
  min-width: 0;
  flex-direction: column;
  align-content: center;
  align-items: center;

  & .dropdown {
    display: flex;
    flex-direction: row;
    align-items: center;
    color: var(--color-text-secondary);
    cursor: pointer;

    & .current-view-name {
      padding: 0 var(--spacing-4);
      color: var(--color-text-tertiary);
    }

    & .icon-chevron-up {
      transition: transform var(--animation-duration) var(--animation-easing);

      &.turned {
        transform: rotate(-180deg);
      }
    }
  }

  & .dropdown-menu {
    z-index: 9999;
    display: flex;
    min-width: 300px;
    flex: 1 0 auto;
    flex-direction: column;
    border: var(--stroke-width-md) solid var(--color-stroke-background-primary);
    border-radius: var(--radius-xs);
    filter: drop-shadow(0 4px 20.9px var(--color-shadow));

    & .new-view,
    & .view-list {
      padding: var(--spacing-4) var(--spacing-6);
    }

    & .new-view {
      display: flex;
      flex-direction: row;
      align-items: center;
      justify-content: space-between;
      border-bottom: var(--stroke-width-md) solid var(--color-stroke-background-primary);
      background-color: var(--color-background-secondary);
      border-top-left-radius: var(--radius-xs);
      border-top-right-radius: var(--radius-xs);
      color: var(--color-text-tertiary);
      cursor: pointer;

      &:is(:last-child) {
        border-bottom: 0;
        border-bottom-left-radius: var(--radius-xs);
        border-bottom-right-radius: var(--radius-xs);
      }
    }

    & .view-list {
      max-height: 20dvh;
      background-color: var(--color-background-primary);
    }
  }
}
</style>
