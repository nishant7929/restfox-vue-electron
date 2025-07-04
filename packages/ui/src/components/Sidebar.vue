<template>
    <div class="sidebar-filter">
        <input type="search" placeholder="Filter" spellcheck="false" v-model="collectionFilter">
        <div style="cursor: pointer; align-self: center; padding-left: 0.5rem; padding-right: 0.5rem;" @click="handleSidebarCreateButton" title="Create New Request / Folder" v-if="collectionFilter === ''">
            <i class="fa fa-plus-circle"></i>
        </div>
        <div style="cursor: pointer; align-self: center; padding-left: 0.5rem; padding-right: 0.5rem;" @click="collapseFolders" title="Collapse Folders" v-if="collectionFilter === ''">
            <i class="fa fa-collapse"></i>
            <i class="fa fa-minus"></i>
        </div>
    </div>
    <div class="sidebar-list-container" @contextmenu.prevent="handleSidebarEmptyAreaContextMenu">
        <div class="sidebar-list">
            <template v-for="sidebarItem in sidebarItems" :key="sidebarItem._id">
                <sidebar-item :sidebar-item="sidebarItem" />
            </template>
        </div>
        <div v-if="sidebarItems.length === 0" class="sidebar-empty-message">
            <template v-if="collectionFilter === ''">
                You can right click here to create a new request or a new folder
            </template>
            <template v-else>
                No results found
            </template>
        </div>
    </div>
    <ContextMenu :options="options" :element="sidebarContextMenuElement" v-model:show="showContextMenu" @click="handleClick" :x="contextMenuX" :y="contextMenuY" :x-offset="20" />
    <AddRequestModal v-model:showModal="addRequestModalShow" :parent-id="addRequestModalParentId" />
    <AddGraphQLRequestModal v-model:showModal="addGraphQLRequestModalShow" :parent-id="addRequestModalParentId" />
    <AddSocketModal v-model:showModal="addSocketModalShow" :parent-id="addSocketModalParentId" />
    <AddFolderModal v-model:showModal="addFolderModalShow" :parent-id="addFolderModalParentId" />
    <EnvironmentModal v-model:showModal="environmentModalShow" :collection-item="environmentModalCollectionItem" />
    <PluginManagerModal v-model:showModal="pluginManagerShow" :collection-item="pluginManagerCollectionItem" />
    <SettingsModal :show-modal="settingsModalShow" :collection-item="settingsModalCollectionItem" @update:collection-item="updateCollectionItem" />
    <DuplicateCollectionItemModal v-model:showModal="showDuplicateCollectionItemModal" :collection-item-to-duplicate="collectionItemToDuplicate" />
    <GenerateCodeModal v-model:showModal="generateCodeModalShow" :collection-item="generateCodeModalCollectionItem" />
</template>

<script>
import { toRaw } from 'vue'
import SidebarItem from './SidebarItem.vue'
import ContextMenu from './ContextMenu.vue'
import AddRequestModal from './modals/AddRequestModal.vue'
import AddSocketModal from './modals/AddSocketModal.vue'
import AddFolderModal from './modals/AddFolderModal.vue'
import EnvironmentModal from './modals/EnvironmentModal.vue'
import PluginManagerModal from './modals/PluginManagerModal.vue'
import SettingsModal from './modals/SidebarSettingsModal.vue'
import DuplicateCollectionItemModal from './modals/DuplicateCollectionItemModal.vue'
import GenerateCodeModal from './modals/GenerateCodeModal.vue'
import { mapState } from 'vuex'
import { flattenTree, exportRestfoxCollection, generateNewIdsForTree, deepClone } from '@/helpers'
import { generateCode } from '@/utils/generate-code'
import AddGraphQLRequestModal from '@/components/modals/AddGraphQLRequestModal.vue'

export default {
    components: {
        SidebarItem,
        ContextMenu,
        AddRequestModal,
        AddGraphQLRequestModal,
        AddSocketModal,
        AddFolderModal,
        EnvironmentModal,
        PluginManagerModal,
        SettingsModal,
        DuplicateCollectionItemModal,
        GenerateCodeModal,
    },
    data() {
        return {
            showContextMenu: false,
            addRequestModalShow: false,
            addGraphQLRequestModalShow: false,
            addRequestModalParentId: null,
            addSocketModalShow: false,
            addSocketModalParentId: null,
            addFolderModalShow: false,
            addFolderModalParentId: null,
            contextMenuX: null,
            contextMenuY: null,
            enableOptionsForEmptyContextMenu: false,
            environmentModalShow: false,
            environmentModalCollectionItem: null,
            settingsModalShow: false,
            settingsModalCollectionItem: null,
            draggedSidebarElement: null,
            sidebarItemCursorPosition: 'below',
            showDuplicateCollectionItemModal: false,
            collectionItemToDuplicate: null,
            pluginManagerCollectionItem: null,
            pluginManagerShow: false,
            generateCodeModalCollectionItem: null,
            generateCodeModalShow: false,
            createNewList: [
                {
                    'type': 'option',
                    'label': 'Create',
                    'icon': 'fa fa-plus',
                    'disabled': true,
                    'class': 'text-with-line'
                },
                {
                    'type': 'option',
                    'label': 'HTTP Request',
                    'value': 'New Request',
                    'icon': 'fa fa-plus-circle',
                    'class': 'context-menu-item-with-left-padding'
                },
                {
                    'type': 'option',
                    'label': 'GraphQL Request',
                    'value': 'New GraphQL Request',
                    'icon': 'fa fa-plus-circle',
                    'class': 'context-menu-item-with-left-padding'
                },
                {
                    'type': 'option',
                    'label': 'WebSocket',
                    'value': 'New Socket',
                    'icon': 'fa fa-plus-circle',
                    'class': 'context-menu-item-with-left-padding'
                },
                {
                    'type': 'option',
                    'label': 'New Folder',
                    'value': 'New Folder',
                    'icon': 'fa fa-folder',
                    'class': 'context-menu-item-with-left-padding'
                },],
            itemProperties: [
                {
                    'type': 'option',
                    'label': '',
                    'icon': 'fa fa-cog',
                    'disabled': true,
                    'class': 'text-with-line'
                },
                {
                    'type': 'option',
                    'label': 'Properties',
                    'value': 'Properties',
                    'icon': 'fa fa-wrench',
                    'class': 'context-menu-item-with-left-padding'
                }
            ]
        }
    },
    computed: {
        sidebarItems() {
            return this.$store.getters.collectionTreeFiltered
        },
        collectionFilter: {
            get() {
                return this.$store.state.collectionFilter
            },
            set(value) {
                this.$store.commit('setCollectionFilter', value)
            }
        },
        ...mapState(['activeSidebarItemForContextMenu', 'sidebarContextMenuElement']),
        options() {
            if(this.enableOptionsForEmptyContextMenu) {
                return [...this.createNewList,
                    {
                        'type': 'option',
                        'label': 'Import',
                        'icon': 'fa fa-file-import',
                        'disabled': true,
                        'class': 'text-with-line'
                    },
                    {
                        'type': 'option',
                        'label': 'Import',
                        'value': 'Import Into',
                        'icon': 'fa fa-file-import',
                        'class': 'context-menu-item-with-left-padding'
                    },
                    {
                        'type': 'option',
                        'label': 'cURL Import',
                        'value': 'cURL Import',
                        'icon': 'fa fa-terminal',
                        'class': 'context-menu-item-with-left-padding'
                    }
                ]
            }

            if(this.activeSidebarItemForContextMenu === null) {
                return []
            }

            if(this.activeSidebarItemForContextMenu._type === 'request' || this.activeSidebarItemForContextMenu._type === 'socket') {
                let contextMenuOptions = [
                    ...this.getCommonActions(this.activeSidebarItemForContextMenu._type),
                    ...this.itemProperties
                ]

                if(this.activeSidebarItemForContextMenu._type === 'socket') {
                    const optionsToRemove = [
                        'Copy as Curl',
                        'Generate Code',
                        'Plugins'
                    ]
                    contextMenuOptions = contextMenuOptions.filter(option => !optionsToRemove.includes(option.value))
                }

                return contextMenuOptions
            }

            if(this.activeSidebarItemForContextMenu._type === 'request_group') {
                return [
                    ...this.createNewList,
                    {
                        'type': 'option',
                        'label': 'Import',
                        'icon': 'fa fa-file-import',
                        'disabled': true,
                        'class': 'text-with-line'
                    },
                    {
                        'type': 'option',
                        'label': 'Import Into',
                        'value': 'Import Into',
                        'icon': 'fa fa-file-import',
                        'class': 'context-menu-item-with-left-padding'
                    },
                    {
                        'type': 'option',
                        'label': 'cURL Import',
                        'value': 'cURL Import',
                        'icon': 'fa fa-terminal',
                        'class': 'context-menu-item-with-left-padding'
                    },
                    ...this.getCommonActions(this.activeSidebarItemForContextMenu._type),
                    ...this.itemProperties
                ]
            }

            return []
        }
    },
    watch: {
        activeSidebarItemForContextMenu() {
            if(this.activeSidebarItemForContextMenu) {
                this.showContextMenu = true
            }
        },
        showContextMenu() {
            if(this.showContextMenu === false) {
                this.$store.commit('clearActiveSidebarItemForContextMenu')
                this.contextMenuX = null
                this.contextMenuY = null
                this.enableOptionsForEmptyContextMenu = false
            }
        }
    },
    methods: {
        async handleClick(clickedSidebarItem) {
            if(clickedSidebarItem === 'Delete') {
                const collectionItemToDelete = deepClone(this.activeSidebarItemForContextMenu)
                if(await window.createConfirm('Are you sure?')) {
                    await this.$store.dispatch('deleteCollectionItem', collectionItemToDelete)
                }
            }

            if(clickedSidebarItem === 'Duplicate') {
                this.collectionItemToDuplicate = structuredClone(toRaw(this.activeSidebarItemForContextMenu))
                this.showDuplicateCollectionItemModal = true
            }

            if(clickedSidebarItem === 'Export') {
                let collectionItemToExport = deepClone(this.activeSidebarItemForContextMenu)
                collectionItemToExport.parentId = null

                collectionItemToExport = [collectionItemToExport]

                // if the workspace is a file workspace, we need to generate new ids for the collection
                // as ids are just file paths in the case of file workspaces
                // we don't want to leak the file paths in the exported collection
                if (this.$store.state.activeWorkspace._type === 'file') {
                    generateNewIdsForTree(collectionItemToExport)
                }

                const collection = flattenTree(collectionItemToExport)
                for(const item of collection) {
                    item.plugins = this.$store.state.plugins.workspace.filter(plugin => plugin.collectionId === item._id)
                }
                exportRestfoxCollection(collection)
            }

            if(clickedSidebarItem === 'Copy as Curl') {
                const request = deepClone(this.activeSidebarItemForContextMenu)
                const { environment, parentHeaders, parentAuthentication } = await this.$store.dispatch('getEnvironmentForRequest', { collectionItem: request })
                try {
                    const curlCommand = await generateCode(request, environment, parentHeaders, parentAuthentication, 'shell', 'curl')
                    await navigator.clipboard.writeText(curlCommand)
                    this.$toast.success('Copied to clipboard')
                } catch (e) {
                    let errorAdditional = ''
                    if(e.message.includes('Invalid URL')) {
                        errorAdditional = ' : Invalid URL'
                    } else {
                        errorAdditional = ' : ' + e.message
                        console.error(e)
                    }
                    this.$toast.error('Failed to copy to clipboard' + errorAdditional)
                }
            }

            if(clickedSidebarItem === 'Generate Code') {
                this.generateCodeModalCollectionItem = deepClone(this.activeSidebarItemForContextMenu)
                this.generateCodeModalShow = true
            }

            if(clickedSidebarItem === 'New Request') {
                this.addRequestModalParentId = this.activeSidebarItemForContextMenu ? this.activeSidebarItemForContextMenu._id : null
                this.addRequestModalShow = true
            }

            if(clickedSidebarItem === 'New GraphQL Request') {
                this.addRequestModalParentId = this.activeSidebarItemForContextMenu ? this.activeSidebarItemForContextMenu._id : null
                this.addGraphQLRequestModalShow = true
            }

            if(clickedSidebarItem === 'New Socket') {
                this.addSocketModalParentId = this.activeSidebarItemForContextMenu ? this.activeSidebarItemForContextMenu._id : null
                this.addSocketModalShow = true
            }

            if(clickedSidebarItem === 'New Folder') {
                this.addFolderModalParentId = this.activeSidebarItemForContextMenu ? this.activeSidebarItemForContextMenu._id : null
                this.addFolderModalShow = true
            }

            if(clickedSidebarItem === 'Environment') {
                this.environmentModalCollectionItem = this.activeSidebarItemForContextMenu
                this.environmentModalShow = true
            }

            if(clickedSidebarItem === 'Plugins') {
                this.pluginManagerCollectionItem = deepClone(this.activeSidebarItemForContextMenu)
                this.pluginManagerShow = true
            }

            if(clickedSidebarItem === 'Import Into') {
                this.$store.commit('showImportModalSelectedRequestGroupId', this.activeSidebarItemForContextMenu ? this.activeSidebarItemForContextMenu._id : null)
                this.$store.commit('showImportModal', true)
            }

            if(clickedSidebarItem === 'cURL Import') {
                this.$store.commit('showImportModalSelectedRequestGroupId', this.activeSidebarItemForContextMenu ? this.activeSidebarItemForContextMenu._id : null)
                this.$store.commit('showImportAsCurlModal', true)
            }

            if(clickedSidebarItem === 'Properties') {
                this.settingsModalCollectionItem = deepClone(this.activeSidebarItemForContextMenu)
                this.settingsModalShow = true
            }

            this.showContextMenu = false
        },
        handleSidebarEmptyAreaContextMenu(event) {
            if(this.collectionFilter !== '') {
                return
            }

            if(event.target.classList.contains('sidebar-list-container') === false && event.target.classList.contains('sidebar-empty-message') === false) {
                return
            }
            this.contextMenuX = event.pageX
            this.contextMenuY = event.pageY
            this.enableOptionsForEmptyContextMenu = true
            this.showContextMenu = true
        },
        handleSidebarCreateButton(event) {
            this.contextMenuX = event.target.getBoundingClientRect().left
            this.contextMenuY = event.target.getBoundingClientRect().top + event.target.getBoundingClientRect().height
            this.enableOptionsForEmptyContextMenu = true
            this.showContextMenu = true
        },
        getDropTargetElementContainer(event) {
            let container
            const sidebarItem = event.target.closest('.sidebar-item')
            const sidebarListContainer = event.target.closest('.sidebar-list-container')
            if (sidebarItem) {
                container = {
                    type: 'sidebar-item',
                    element: sidebarItem
                }
            } else if (sidebarListContainer) {
                const innerList = sidebarListContainer.querySelector('.sidebar-list')
                if (!innerList) {
                    return
                }
                container = {
                    type: 'sidebar-list',
                    element: innerList
                }
            } else {
                return null
            }
            return container
        },
        dragStart(event) {
            if(this.collectionFilter) { // disable drag functionality if collection is being filtered
                return
            }
            this.draggedSidebarElement = event.target.closest('.sidebar-item')
            if(!this.draggedSidebarElement) {
                return
            }
            this.draggedSidebarElement.style.backgroundColor = 'var(--background-color)'
            this.draggedSidebarElement.style.opacity = '0.5'
        },
        dragEnd() {
            if(!this.draggedSidebarElement) {
                return
            }
            this.draggedSidebarElement.style.backgroundColor = ''
            this.draggedSidebarElement.style.opacity = ''
            this.draggedSidebarElement = null
        },

        dragOver(event) {
            if(!this.draggedSidebarElement) {
                return
            }
            const container = this.getDropTargetElementContainer(event)
            if(!container) {
                return
            }

            const elementToDropOn = container.element
            const rect = elementToDropOn.getBoundingClientRect()
            const offset = rect.top + document.body.scrollTop
            const elementHeight = parseFloat(getComputedStyle(elementToDropOn, null).height.replace('px', ''))
            const y = event.pageY
            const location = Math.abs(offset - y)
            if (location < (elementHeight / 2)) {
                this.sidebarItemCursorPosition = 'top'
                elementToDropOn.style.borderTop = '1px dashed var(--text-color)'
                elementToDropOn.style.borderBottom = ''
                elementToDropOn.style.backgroundColor = ''
            } else {
                this.sidebarItemCursorPosition = 'bottom'
                if(elementToDropOn.dataset.type === 'request_group') {
                    elementToDropOn.style.borderTop = ''
                    elementToDropOn.style.borderBottom = ''
                    elementToDropOn.style.backgroundColor = 'var(--drop-target-background-color)'
                } else {
                    elementToDropOn.style.borderTop = ''
                    elementToDropOn.style.borderBottom = '1px dashed var(--text-color)'
                    elementToDropOn.style.backgroundColor = ''
                }
            }
            event.preventDefault()
        },
        dragLeave(event) {
            if(!this.draggedSidebarElement) {
                return
            }
            const container = this.getDropTargetElementContainer(event)
            if(!container) {
                return
            }
            const elementToDropOn = container.element
            if(elementToDropOn) {
                elementToDropOn.style.borderBottom = ''
                elementToDropOn.style.borderTop = ''
                elementToDropOn.style.backgroundColor = ''
            }
        },
        drop(event) {
            if(!this.draggedSidebarElement) {
                return
            }
            event.preventDefault()
            const container = this.getDropTargetElementContainer(event)
            if(!container) {
                return
            }

            const elementToDropOn = container.element
            if(elementToDropOn) {
                elementToDropOn.style.borderTop = ''
                elementToDropOn.style.borderBottom = ''
                elementToDropOn.style.backgroundColor = ''

                this.$store.dispatch('reorderCollectionItem', {
                    from: {
                        parentId: this.draggedSidebarElement.dataset.parentId,
                        id: this.draggedSidebarElement.dataset.id
                    },
                    to: {
                        parentId: elementToDropOn.dataset.parentId,
                        id: elementToDropOn.dataset.id,
                        type: elementToDropOn.dataset.type ?? container.type,
                        cursorPosition: this.sidebarItemCursorPosition
                    }
                })

                this.draggedSidebarElement.style.backgroundColor = ''
                this.draggedSidebarElement.style.opacity = ''
                this.draggedSidebarElement = null
            }
        },
        async updateCollectionItem(collectionItem) {
            const result = await this.$store.dispatch('updateCollectionItem', {
                collectionId: collectionItem._id,
                name: collectionItem.name,
                parentId: collectionItem.parentId,
                headers: toRaw(collectionItem.headers),
                authentication: toRaw(collectionItem.authentication),
            })

            if(result.error) {
                return
            }

            this.settingsModalShow = false
        },
        collapseFolders() {
            this.$store.dispatch('collapseFolders', this.sidebarItems)
        },
        getCommonActions(type) {
            const startIndex = 3
            const contextMenuItemClass = 'context-menu-item-with-left-padding'

            let commonActions = [
                {
                    'type': 'option',
                    'label': 'Actions',
                    'icon': 'fa fa-cog',
                    'disabled': true,
                    'class': 'text-with-line'
                },
                {
                    'type': 'option',
                    'label': 'Duplicate',
                    'value': 'Duplicate',
                    'icon': 'fa fa-copy',
                    'class': contextMenuItemClass
                },
                {
                    'type': 'option',
                    'label': 'Export',
                    'value': 'Export',
                    'icon': 'fa fa-file-export',
                    'class': contextMenuItemClass
                },
                {
                    'type': 'option',
                    'label': 'Plugins',
                    'value': 'Plugins',
                    'icon': 'fa fa-plug',
                    'class': contextMenuItemClass
                },
                {
                    'type': 'option',
                    'label': 'Delete',
                    'value': 'Delete',
                    'icon': 'fa fa-trash',
                    'class': contextMenuItemClass
                },
            ]

            if(type === 'request_group'){
                commonActions.splice(startIndex, 0,{
                    'type': 'option',
                    'label': 'Environment',
                    'value': 'Environment',
                    'icon': 'fa fa-code',
                    'class': contextMenuItemClass
                })
            } else {
                commonActions.splice(startIndex, 0,...[{
                    'type': 'option',
                    'label': 'Copy as Curl',
                    'value': 'Copy as Curl',
                    'icon': 'fa fa-copy',
                    'class': contextMenuItemClass
                },
                {
                    'type': 'option',
                    'label': 'Generate Code',
                    'value': 'Generate Code',
                    'icon': 'fa fa-code',
                    'class': contextMenuItemClass
                }])
            }
            return commonActions
        }
    },
    mounted() {
        document.addEventListener('dragstart', this.dragStart)
        document.addEventListener('dragend', this.dragEnd)
        document.addEventListener('dragover', this.dragOver)
        document.addEventListener('dragleave', this.dragLeave)
        document.addEventListener('drop', this.drop)
    },
    unmounted() {
        document.removeEventListener('dragstart', this.dragStart)
        document.removeEventListener('dragend', this.dragEnd)
        document.removeEventListener('dragover', this.dragOver)
        document.removeEventListener('dragleave', this.dragLeave)
        document.removeEventListener('drop', this.drop)
    },
}
</script>

<style>
.sidebar .sidebar-filter {
    display: flex;
    border-bottom: 1px solid var(--default-border-color);
}

.sidebar .sidebar-filter > input  {
    width: 100%;
    padding: 5px;
    outline: 0;
    border: 0;
}

.sidebar .sidebar-list-container {
    height: 100%;
    overflow: auto;
    padding-top: 0.5rem;
    padding-bottom: 1rem;
    background-color: var(--cm-gutter-background-color);
}

.sidebar .sidebar-item {
    padding-top: 0.4rem;
    padding-bottom: 0.4rem;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    display: flex;
    align-items: center;
    border: 1px solid transparent;
    position: relative;
    border-left: 2px solid transparent;
}

.sidebar .sidebar-item-active {
    background-color: var(--sidebar-item-active-color);
    border: 1px solid var(--default-border-color);
    border-right: 0;
    border-radius: var(--default-border-radius);
    border-left: 2px solid #ff6c37;
}

.sidebar .sidebar-item:hover {
    background-color: var(--sidebar-item-active-color);
}

.sidebar .sidebar-list > .sidebar-item {
    padding-left: calc(1rem * 0.6);
}

/* handle nested sidebar lists */
.sidebar .sidebar-list > .sidebar-list {
    padding-left: calc(1rem * 1.2);
}

.sidebar .sidebar-item-method {
    min-width: 1.9rem;
    max-width: 1.9rem;
    margin-right: 0.2rem;
    overflow: hidden;
    font-size: 0.7rem;
}

.sidebar .sidebar-item *:not(.request-group-toggle) {
    pointer-events: none;
}

.sidebar .sidebar-item .clickable-context-menu {
    display: none;
}

.sidebar .sidebar-item:hover .clickable-context-menu {
    display: block;
    position: absolute;
    right: 0;
}

.sidebar .sidebar-item .clickable-context-menu i {
    pointer-events: auto;
    cursor: pointer;
    height: 100%;
    padding-left: 1rem;
    padding-right: 1rem;
    color: var(--text-color);
}

.sidebar-empty-message {
    padding-left: 1rem;
    padding-right: 1rem;
    padding-top: 0.5rem;
}
</style>
