<script lang="ts" setup>
import { nextTick } from '@vue/runtime-core'
import { message } from 'ant-design-vue'
import type { BaseType, ProjectType, TableType } from 'nocodb-sdk'
import { LoadingOutlined } from '@ant-design/icons-vue'
import { useTitle } from '@vueuse/core'
import {
  NcProjectType,
  ProjectInj,
  ProjectRoleInj,
  ToggleDialogInj,
  extractSdkResponseErrorMsg,
  isElementInvisible,
  openLink,
  storeToRefs,
  useProjects,
} from '#imports'
import type { NcProject } from '#imports'
import { useNuxtApp } from '#app'

const indicator = h(LoadingOutlined, {
  class: '!text-gray-400',
  style: {
    fontSize: '0.85rem',
  },
  spin: true,
})

const router = useRouter()
const route = router.currentRoute

const { setMenuContext, openRenameTableDialog, duplicateTable, contextMenuTarget } = inject(TreeViewInj)!

const project = inject(ProjectInj)!

const projectsStore = useProjects()

const { loadProject, loadProjects, createProject: _createProject, updateProject, getProjectMetaInfo } = projectsStore
const { projects } = storeToRefs(projectsStore)

const { loadProjectTables } = useTablesStore()
const { activeTable } = storeToRefs(useTablesStore())

const { appInfo, navigateToProject } = useGlobal()

useTabs()

const editMode = ref(false)

const tempTitle = ref('')

const { t } = useI18n()

const input = ref<HTMLInputElement>()

const { isUIAllowed } = useUIPermission()

const projectRole = inject(ProjectRoleInj)

const { activeProjectId } = storeToRefs(useProjects())

const { projectUrl } = useProject()

const toggleDialog = inject(ToggleDialogInj, () => {})

const { $e } = useNuxtApp()

const isOptionsOpen = ref(false)
const isBasesOptionsOpen = ref<Record<string, boolean>>({})

const activeKey = ref<string[]>([])
const [searchActive] = useToggle()
const filterQuery = ref('')
const keys = ref<Record<string, number>>({})
const isTableDeleteDialogVisible = ref(false)
const isProjectDeleteDialogVisible = ref(false)

// If only project is open, i.e in case of docs, project view is open and not the page view
const projectViewOpen = computed(() => {
  const routeNameSplit = String(route.value?.name).split('projectId-index-index')
  if (routeNameSplit.length <= 1) return false

  const routeNameAfterProjectView = routeNameSplit[routeNameSplit.length - 1]
  return routeNameAfterProjectView.split('-').length === 2 || routeNameAfterProjectView.split('-').length === 1
})

const enableEditMode = () => {
  editMode.value = true
  tempTitle.value = project.value.title!
  nextTick(() => {
    input.value?.focus()
    input.value?.select()
    input.value?.scrollIntoView()
  })
}

const updateProjectTitle = async () => {
  if (!tempTitle.value) return

  try {
    await updateProject(project.value.id!, {
      title: tempTitle.value,
    })
    editMode.value = false
    tempTitle.value = ''

    $e('a:project:rename')

    useTitle(`${project.value?.title}`)
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
}

const { copy } = useCopy(true)

const copyProjectInfo = async () => {
  try {
    if (
      await copy(
        Object.entries(await getProjectMetaInfo(project.value.id!)!)
          .map(([k, v]) => `${k}: **${v}**`)
          .join('\n'),
      )
    ) {
      // Copied to clipboard
      message.info(t('msg.info.copiedToClipboard'))
    }
  } catch (e: any) {
    console.error(e)
    message.error(e.message)
  }
}

defineExpose({
  enableEditMode,
})

const setIcon = async (icon: string, project: ProjectType) => {
  try {
    const meta = {
      ...((project.meta as object) || {}),
      icon,
    }

    projectsStore.updateProject(project.id!, { meta: JSON.stringify(meta) })

    $e('a:project:icon:navdraw', { icon })
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
}

function openTableCreateDialog(baseIndex?: number | undefined) {
  $e('c:table:create:navdraw')

  const isOpen = ref(true)
  let baseId = project.value!.bases?.[0].id
  if (typeof baseIndex === 'number') {
    baseId = project.value!.bases?.[baseIndex].id
  }

  if (!baseId || !project.value?.id) return

  const { close } = useDialog(resolveComponent('DlgTableCreate'), {
    'modelValue': isOpen,
    baseId, // || bases.value[0].id,
    'projectId': project.value!.id,
    'onCreate': closeDialog,
    'onUpdate:modelValue': () => closeDialog(),
  })

  function closeDialog(table?: TableType) {
    isOpen.value = false

    if (!table) return

    if (!activeKey.value || !activeKey.value.includes(`collapse-${baseId}`)) {
      activeKey.value.push(`collapse-${baseId}`)
    }

    // TODO: Better way to know when the table node dom is available
    setTimeout(() => {
      const newTableDom = document.querySelector(`[data-table-id="${table.id}"]`)
      if (!newTableDom) return

      // Verify that table node is not in the viewport
      if (isElementInvisible(newTableDom)) {
        // Scroll to the table node
        newTableDom?.scrollIntoView({ behavior: 'smooth' })
      }
    }, 1000)

    close(1000)
  }
}

const isAddNewProjectChildEntityLoading = ref(false)
const addNewProjectChildEntity = async () => {
  if (isAddNewProjectChildEntityLoading.value) return

  isAddNewProjectChildEntityLoading.value = true
  try {
    openTableCreateDialog()

    if (!project.value.isExpanded) {
      project.value.isExpanded = true
    }
  } finally {
    isAddNewProjectChildEntityLoading.value = false
  }
}

// todo: temp
const isSharedBase = ref(false)

const onProjectClick = async (project: NcProject, ignoreNavigation?: boolean, toggleIsExpanded?: boolean) => {
  if (!project) {
    return
  }

  if (toggleIsExpanded) {
    project.isExpanded = !project.isExpanded
  } else {
    project.isExpanded = true
  }

  const isProjectPopulated = projectsStore.isProjectPopulated(project.id!)

  let isSharedBase = false
  // if shared base ignore navigation
  if (route.value.params.typeOrId === 'base') {
    isSharedBase = true
  }

  if (!isProjectPopulated) project.isLoading = true

  if (!ignoreNavigation) {
    await navigateTo(
      projectUrl({
        id: project.id!,
        type: 'database',
        isSharedBase,
      }),
    )
  }

  if (!isProjectPopulated) {
    await loadProject(project.id!)
    await loadProjectTables(project.id!)
  }

  if (!isProjectPopulated) {
    const updatedProject = projects.value.get(project.id!)!
    updatedProject.isLoading = false
  }
}

function openErdView(base: BaseType) {
  navigateTo(`/nc/${base.project_id}/erd/${base.id}`)
}

async function openProjectErdView(_project: ProjectType) {
  if (!_project.id) return

  if (!projectsStore.isProjectPopulated(_project.id)) {
    await loadProject(_project.id)
  }

  const project = projects.value.get(_project.id)

  const base = project?.bases?.[0]
  if (!base) return
  navigateTo(`/nc/${base.project_id}/erd/${base.id}`)
}

const reloadTables = async () => {
  $e('a:table:refresh:navdraw')

  // await loadTables()
}

const contextMenuBase = computed(() => {
  if (contextMenuTarget.type === 'base') {
    return contextMenuTarget.value
  } else if (contextMenuTarget.type === 'table') {
    const base = project.value?.bases?.find((b) => b.id === contextMenuTarget.value.base_id)
    if (base) return base
  }
  return null
})

watch(
  () => activeTable.value?.id,
  async () => {
    if (!activeTable.value) return

    const baseId = activeTable.value.base_id
    if (!baseId) return

    if (!activeKey.value.includes(`collapse-${baseId}`)) {
      activeKey.value.push(`collapse-${baseId}`)
    }
  },
  {
    immediate: true,
  },
)

onKeyStroke('Escape', () => {
  if (isOptionsOpen.value) {
    isOptionsOpen.value = false
  }

  for (const key of Object.keys(isBasesOptionsOpen.value)) {
    isBasesOptionsOpen.value[key] = false
  }
})

const isDuplicateDlgOpen = ref(false)
const selectedProjectToDuplicate = ref()

const duplicateProject = (project: ProjectType) => {
  selectedProjectToDuplicate.value = project
  isDuplicateDlgOpen.value = true
}
const { $jobs } = useNuxtApp()

const DlgProjectDuplicateOnOk = async (jobData: { id: string; project_id: string }) => {
  await loadProjects('workspace')

  $jobs.subscribe({ id: jobData.id }, undefined, async (status: string) => {
    if (status === JobStatus.COMPLETED) {
      await loadProjects('workspace')

      const project = projects.value.get(jobData.project_id)

      // open project after duplication
      if (project) {
        await navigateToProject({
          projectId: project.id,
          type: project.type,
        })
      }
    } else if (status === JobStatus.FAILED) {
      message.error('Failed to duplicate project')
      await loadProjects('workspace')
    }
  })

  $e('a:project:duplicate')
}
</script>

<template>
  <a-dropdown :trigger="['contextmenu']" overlay-class-name="nc-dropdown-tree-view-context-menu">
    <div
      class="mx-1 nc-project-sub-menu rounded-md"
      :class="{ active: project.isExpanded }"
      :data-testid="`nc-sidebar-project-${project.title}`"
      :data-project-id="project.id"
    >
      <div class="flex items-center gap-0.75 py-0.25 cursor-pointer" @contextmenu="setMenuContext('project', project)">
        <div
          ref="projectNodeRefs"
          :class="{
            'bg-primary-selected active': activeProjectId === project.id && projectViewOpen,
            'hover:bg-gray-200': !(activeProjectId === project.id && projectViewOpen),
          }"
          :data-testid="`nc-sidebar-project-title-${project.title}`"
          class="project-title-node h-7.25 flex-grow rounded-md group flex items-center w-full"
        >
          <div
            class="nc-sidebar-expand ml-0.75 min-h-5.75 min-w-5.75 px-1.5 text-gray-500 hover:(hover:bg-gray-500 hover:bg-opacity-15 !text-black) rounded-md relative"
            @click="onProjectClick(project, true, true)"
          >
            <PhTriangleFill
              class="absolute top-2.25 left-2 invisible group-hover:visible cursor-pointer transform transition-transform duration-500 h-1.5 w-1.75 rotate-90"
              :class="{ '!rotate-180': project.isExpanded, '!visible': isOptionsOpen }"
            />
          </div>

          <div class="flex items-center mr-1" @click="onProjectClick(project)">
            <div class="flex items-center select-none w-6 h-full">
              <a-spin
                v-if="project.isLoading"
                class="nc-sidebar-icon !flex !flex-row !items-center !my-0.5 !mx-1.5 w-8"
                :indicator="indicator"
              />

              <LazyGeneralEmojiPicker
                :key="project.meta?.icon"
                :emoji="project.meta?.icon"
                :readonly="true"
                size="small"
                @emoji-selected="setIcon($event, project)"
              >
                <template #default>
                  <GeneralProjectIcon :type="project.type" />
                </template>
              </LazyGeneralEmojiPicker>
            </div>
          </div>

          <input
            v-if="editMode"
            ref="input"
            v-model="tempTitle"
            class="flex-grow leading-1 outline-0 ring-none capitalize !text-inherit !bg-transparent w-4/5"
            :class="{ 'text-black font-semibold': activeProjectId === project.id && projectViewOpen }"
            @click.stop
            @keyup.enter="updateProjectTitle"
            @keyup.esc="updateProjectTitle"
            @blur="updateProjectTitle"
          />
          <span
            v-else
            class="capitalize text-ellipsis overflow-hidden select-none"
            :style="{ wordBreak: 'keep-all', whiteSpace: 'nowrap', display: 'inline' }"
            :class="{ 'text-black font-semibold': activeProjectId === project.id && projectViewOpen }"
            @click="onProjectClick(project)"
          >
            {{ project.title }}
          </span>
          <div :class="{ 'flex flex-grow h-full': !editMode }" @click="onProjectClick(project)"></div>

          <a-dropdown v-if="isUIAllowed('tableCreate', false, projectRole)" v-model:visible="isOptionsOpen" trigger="click">
            <MdiDotsHorizontal
              class="min-w-5.75 min-h-5.75 px-0.5 py-0.5 mr-0.25 !ring-0 focus:!ring-0 !focus:border-0 !focus:outline-0 opacity-0 group-hover:(opacity-100) hover:text-black text-gray-600 rounded-md hover:(bg-gray-500 bg-opacity-15)"
              :class="{ '!text-black !opacity-100': isOptionsOpen }"
              data-testid="nc-sidebar-context-menu"
              @click.stop
            />
            <template #overlay>
              <a-menu
                class="nc-scrollbar-md"
                :style="{
                  maxHeight: '70vh',
                  overflow: 'overlay',
                }"
                @click="isOptionsOpen = false"
              >
                <template v-if="!isSharedBase">
                  <a-menu-item @click="enableEditMode">
                    <div class="nc-project-menu-item group">
                      <GeneralIcon icon="edit" class="group-hover:text-black" />
                      {{ $t('general.edit') }}
                    </div>
                  </a-menu-item>

                  <!-- Copy Project Info -->
                  <a-menu-item v-if="!isEeUI" key="copy">
                    <div v-e="['c:navbar:user:copy-proj-info']" class="nc-project-menu-item group" @click.stop="copyProjectInfo">
                      <GeneralIcon icon="copy" class="group-hover:text-black" />
                      {{ $t('activity.account.projInfo') }}
                    </div>
                  </a-menu-item>
                  <a-menu-item v-if="isUIAllowed('duplicateProject', true, projectRole)" @click="duplicateProject(project)">
                    <div class="nc-menu-item-wrapper">
                      <GeneralIcon icon="duplicate" class="text-gray-700" />
                      {{ $t('general.duplicate') }} {{ $t('objects.project') }}
                    </div>
                  </a-menu-item>

                  <a-menu-divider v-if="false" />

                  <!-- ERD View -->
                  <a-menu-item key="erd" @click="openProjectErdView(project)">
                    <div class="nc-project-menu-item group">
                      <GeneralIcon icon="erd" />
                      Relations
                    </div>
                  </a-menu-item>

                  <!-- Swagger: Rest APIs -->
                  <a-menu-item key="api">
                    <div
                      v-if="isUIAllowed('apiDocs')"
                      v-e="['e:api-docs']"
                      class="nc-project-menu-item group"
                      @click.stop="openLink(`/api/v1/db/meta/projects/${project.id}/swagger`, appInfo.ncSiteUrl)"
                    >
                      <GeneralIcon icon="snippet" class="group-hover:text-black" />
                      {{ $t('activity.account.swagger') }}
                    </div>
                  </a-menu-item>
                </template>
                <!-- Team & Settings -->
                <a-menu-item key="teamAndSettings">
                  <div
                    v-if="isUIAllowed('settings')"
                    v-e="['c:navdraw:project-settings']"
                    class="nc-project-menu-item group"
                    @click="toggleDialog(true, 'teamAndAuth', undefined, project.id)"
                  >
                    <GeneralIcon icon="settings" class="group-hover:text-black" />
                    {{ $t('activity.settings') }}
                  </div>
                </a-menu-item>
                <template v-if="project.bases && project.bases[0]">
                  <DashboardTreeViewNewBaseOptions v-model:project="project" :base="project.bases[0]" />

                  <a-menu-divider />
                </template>

                <a-menu-divider v-if="false" />

                <a-menu-item v-if="isUIAllowed('projectDelete', false, projectRole)" @click="isProjectDeleteDialogVisible = true">
                  <div class="nc-project-menu-item group text-red-500">
                    <GeneralIcon icon="delete" />
                    {{ $t('general.delete') }}
                  </div>
                </a-menu-item>
              </a-menu>
            </template>
          </a-dropdown>

          <div
            v-if="isUIAllowed('tableCreate', false, projectRole)"
            class="min-h-5.75 min-w-5.75 mr-1 flex flex-row items-center justify-center gap-x-2 cursor-pointer hover:(text-black) text-gray-600 text-sm invisible !group-hover:visible rounded-md hover:(bg-gray-500 bg-opacity-15)"
            data-testid="nc-sidebar-add-project-entity"
            :class="{ '!text-black !visible': isAddNewProjectChildEntityLoading, '!visible': isOptionsOpen }"
            @click.stop="addNewProjectChildEntity"
          >
            <div v-if="isAddNewProjectChildEntityLoading" class="flex flex-row items-center">
              <a-spin class="!flex !flex-row !items-center !my-0.5" :indicator="indicator" />
            </div>
            <MdiPlus v-else class="min-w-5 min-h-5 py-0.25" />
          </div>
        </div>
      </div>

      <div
        v-if="project.id && !project.isLoading"
        key="g1"
        class="overflow-x-hidden transition-max-height"
        :class="{ 'max-h-0': !project.isExpanded }"
      >
        <template v-if="project && project?.bases">
          <div class="flex-1 overflow-y-auto overflow-x-hidden flex flex-col" :class="{ 'mb-[20px]': isSharedBase }">
            <div v-if="project?.bases?.[0]?.enabled" class="flex-1">
              <div class="transition-height duration-200">
                <DashboardTreeViewNewTableList :project="project" :base-index="0" />
              </div>
            </div>

            <div v-if="project?.bases?.slice(1).filter((el) => el.enabled)?.length" class="transition-height duration-200">
              <div class="border-none sortable-list">
                <div v-for="(base, baseIndex) of project.bases" :key="`base-${base.id}`">
                  <template v-if="baseIndex === 0"></template>
                  <a-collapse
                    v-else-if="base && base.enabled"
                    v-model:activeKey="activeKey"
                    class="!mx-0 !px-0 nc-sidebar-base-node"
                    :class="[{ hidden: searchActive && !!filterQuery }]"
                    expand-icon-position="left"
                    :bordered="false"
                    ghost
                  >
                    <template #expandIcon="{ isActive }">
                      <div class="flex flex-row items-center -mt-2">
                        <PhTriangleFill
                          class="nc-sidebar-base-node-btns -mt-0.75 invisible cursor-pointer transform transition-transform duration-500 h-1.5 w-1.5 text-gray-500 rotate-90"
                          :class="{ '!rotate-180': isActive }"
                        />
                      </div>
                    </template>
                    <a-collapse-panel :key="`collapse-${base.id}`">
                      <template #header>
                        <div class="min-w-20 w-full flex flex-row">
                          <div
                            v-if="baseIndex === 0"
                            class="base-context flex items-center gap-2 text-gray-800"
                            @contextmenu="setMenuContext('base', base)"
                          >
                            <GeneralBaseLogo :base-type="base.type" />
                            Default
                          </div>
                          <div
                            v-else
                            class="base-context flex flex-grow items-center gap-1.75 text-gray-800 min-w-1/20 max-w-full"
                            @contextmenu="setMenuContext('base', base)"
                          >
                            <GeneralBaseLogo :base-type="base.type" class="min-w-4" />
                            <div
                              :data-testid="`nc-sidebar-project-${base.alias}`"
                              class="flex capitalize text-ellipsis overflow-hidden select-none"
                              :style="{ wordBreak: 'keep-all', whiteSpace: 'nowrap', display: 'inline' }"
                            >
                              {{ base.alias || '' }}
                            </div>
                            <a-tooltip>
                              <template #title>External DB</template>
                              <div>
                                <GeneralIcon icon="info" class="text-gray-400 -mt-0.5 hover:text-gray-700 mr-1" />
                              </div>
                            </a-tooltip>
                          </div>
                          <div
                            v-if="isUIAllowed('tableCreate', false, projectRole)"
                            class="flex flex-row items-center gap-x-0.25 w-12.25"
                          >
                            <a-dropdown
                              :visible="isBasesOptionsOpen[base!.id!]"
                              trigger="click"
                              @update:visible="isBasesOptionsOpen[base!.id!] = $event"
                            >
                              <MdiDotsHorizontal
                                class="min-w-6 min-h-6 mt-0.15 invisible nc-sidebar-base-node-btns !ring-0 focus:!ring-0 !focus:border-0 !focus:outline-0 hover:text-black py-0.25 px-0.5 rounded-md text-gray-600 hover:(bg-gray-400 bg-opacity-20)"
                                :class="{ '!text-black !opacity-100': isBasesOptionsOpen[base!.id!] }"
                                @click.stop="isBasesOptionsOpen[base!.id!] = !isBasesOptionsOpen[base!.id!]"
                              />
                              <template #overlay>
                                <a-menu
                                  class="nc-scrollbar-md"
                                  :style="{
                                    maxHeight: '70vh',
                                    overflow: 'overlay',
                                  }"
                                  @click="isBasesOptionsOpen[base!.id!] = false"
                                >
                                  <!-- ERD View -->
                                  <a-menu-item key="erd" @click="openErdView(base)">
                                    <div class="nc-project-menu-item group">
                                      <GeneralIcon icon="erd" />
                                      Relations
                                    </div>
                                  </a-menu-item>

                                  <DashboardTreeViewNewBaseOptions v-model:project="project" :base="base" />
                                </a-menu>
                              </template>
                            </a-dropdown>

                            <div
                              v-if="isUIAllowed('tableCreate', false, projectRole)"
                              class="flex invisible nc-sidebar-base-node-btns !focus:outline-0 text-gray-600 hover:text-black px-0.35 rounded-md hover:(bg-gray-500 bg-opacity-15) min-h-6 mt-0.15 min-w-6"
                              @click.stop="openTableCreateDialog(baseIndex)"
                            >
                              <component :is="iconMap.plus" class="text-inherit mt-0.25 h-5.5 w-5.5 py-0.5 !focus:outline-0" />
                            </div>
                          </div>
                        </div>
                      </template>
                      <!-- <AddNewTableNode
                        :project="project"
                        :base-index="baseIndex"
                        @open-table-create-dialog="openTableCreateDialog()"
                      /> -->
                      <div
                        ref="menuRefs"
                        :key="`sortable-${base.id}-${base.id && base.id in keys ? keys[base.id] : '0'}`"
                        :nc-base="base.id"
                      >
                        <DashboardTreeViewNewTableList :project="project" :base-index="baseIndex" />
                      </div>
                    </a-collapse-panel>
                  </a-collapse>
                </div>
              </div>
            </div>
          </div>
        </template>
      </div>
    </div>
    <template v-if="!isSharedBase" #overlay>
      <a-menu class="!py-0 rounded text-sm">
        <template v-if="contextMenuTarget.type === 'project' && project.type === 'database'"></template>

        <template v-else-if="contextMenuTarget.type === 'base'"></template>

        <template v-else-if="contextMenuTarget.type === 'table'">
          <a-menu-item v-if="isUIAllowed('table-rename')" @click="openRenameTableDialog(contextMenuTarget.value, true)">
            <div class="nc-project-menu-item">
              <GeneralIcon icon="edit" class="text-gray-700" />
              {{ $t('general.rename') }}
            </div>
          </a-menu-item>

          <a-menu-item
            v-if="isUIAllowed('table-duplicate') && (contextMenuBase?.is_meta || contextMenuBase?.is_local)"
            @click="duplicateTable(contextMenuTarget.value)"
          >
            <div class="nc-project-menu-item">
              <GeneralIcon icon="duplicate" class="text-gray-700" />
              {{ $t('general.duplicate') }}
            </div>
          </a-menu-item>

          <a-menu-item v-if="isUIAllowed('table-delete')" @click="isTableDeleteDialogVisible = true">
            <div class="nc-project-menu-item text-red-600">
              <GeneralIcon icon="delete" />
              {{ $t('general.delete') }}
            </div>
          </a-menu-item>
        </template>

        <template v-else>
          <a-menu-item @click="reloadTables">
            <div class="nc-project-menu-item">
              {{ $t('general.reload') }}
            </div>
          </a-menu-item>
        </template>
      </a-menu>
    </template>
  </a-dropdown>
  <DlgTableDelete
    v-if="contextMenuTarget.value?.id && project?.id"
    v-model:visible="isTableDeleteDialogVisible"
    :table-id="contextMenuTarget.value?.id"
    :project-id="project?.id"
  />
  <DlgProjectDelete v-model:visible="isProjectDeleteDialogVisible" :project-id="project?.id" />
  <DlgProjectDuplicate
    v-if="selectedProjectToDuplicate"
    v-model="isDuplicateDlgOpen"
    :project="selectedProjectToDuplicate"
    :on-ok="DlgProjectDuplicateOnOk"
  />
</template>

<style lang="scss" scoped>
.nc-sidebar-icon {
  @apply ml-0.5 mr-1;
}

:deep(.ant-collapse-header) {
  @apply !mx-0 !pl-8.75 !pr-1 !py-0.75 hover:bg-gray-100 !rounded-md;
}

:deep(.ant-collapse-header:hover .nc-sidebar-base-node-btns) {
  @apply visible;
}

:deep(.ant-dropdown-menu-submenu-title) {
  @apply !py-0;
}
</style>
