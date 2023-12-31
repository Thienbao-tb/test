<script setup lang="ts">
import { ref, reactive, watch } from "vue"
import { useRoute } from "vue-router"
import useLayoutStore from "../store/layout.store"
import { ListMusicAndCollection, CreateCollection, GetCollectionBreadcrumb } from '../util/music'
import { ConvertMidiToJsonFromFile } from "../util/converter"
import MusicFileEntry from "../component/piano/MusicFileEntry.vue"
import PrimaryInput from "../component/input/PrimaryInput.vue"
import IconOnlyButton from "../component/button/IconOnlyButton.vue"
import PrimaryButton from "../component/button/PrimaryButton.vue"
import PrimaryDialog from "../component/layout/PrimaryDialog.vue"
import ContextMenu from "../component/layout/ContextMenu.vue"
import ContextMenuItem from "../component/layout/ContextMenuItem.vue"
import type { FileEntry } from "@tauri-apps/api/fs"

const route = useRoute()
const layoutStore = useLayoutStore()
const message = () => layoutStore.locale.message
let is_loaded = ref(false)
let list_music = ref<FileEntry[]>([])
let list_collection = ref<FileEntry[]>([])
let collection_breadcrumb = ref<FileEntry[]>([])
let section_el = ref<HTMLElement>()
let is_open_context_menu = ref(false)
let search_string = ref("")
let current_path: string | undefined

updateCurrentPath()
loadCollection()

watch(route, () => {
    updateCurrentPath()
    loadCollection()
}, { immediate: true })

function updateCurrentPath() {
    if (route.query.path && typeof route.query.path == "string") {
        current_path = route.query.path
    } else {
        current_path = undefined
    }
}

async function loadCollection(path?: string) {
    if (path) current_path = path
    
    let result = await ListMusicAndCollection(current_path)
    list_collection.value = result.collections
    list_music.value = result.musics
    
    collection_breadcrumb.value = await GetCollectionBreadcrumb(current_path)
    
    is_loaded.value = true
    search_string.value = ""
}

async function importFromMIDI() {
    closeContextMenu()
    await ConvertMidiToJsonFromFile(current_path)
    await loadCollection()
}

function closeContextMenu() {
    is_open_context_menu.value = false
}

let createCollection = reactive({
    is_show: false,
    collection_name: "",

    Open() {
        createCollection.is_show = true
        closeContextMenu()
    },

    Create() {
        CreateCollection(createCollection.collection_name, current_path)
        loadCollection()
        createCollection.collection_name = ""
        createCollection.is_show = false
    }
})
</script>

<template>
<section ref="section_el" class="container-md" :key="route.fullPath">
    <div class="navbar">
        <PrimaryInput
            class="search"
            type="search"
            icon="search"
            v-model="search_string"
            :placeholder="message().music_searchbar_placeholder"
        />

        <IconOnlyButton
            icon="create_new_folder"
            @click="createCollection.Open"
            :title="message().music_create_collection_button"
        />

        <IconOnlyButton
            icon="audio_file"
            @click="importFromMIDI"
            :title="message().music_import_midi_file_button"
        />
    </div>

    <div class="breadcrumb">
        <div>
            <RouterLink to="/music">Home</RouterLink>
        </div>

        <div v-for="entry in collection_breadcrumb" :key="entry.path">
            / <RouterLink :to="'/music?path=' + entry.path">{{ entry.name }}</RouterLink>
        </div>
    </div>

    <div class="list" v-if="is_loaded">
        <div v-for="collection in list_collection" :key="collection.path">
            <MusicFileEntry
                type="collection"
                :name="collection.name || '_'"
                :path="collection.path"
                @on:remove="loadCollection"
                @on:rename="loadCollection"
                v-show="collection.name?.toLowerCase().includes(search_string.toLowerCase())"
            />
        </div>
        
        <div v-for="music in list_music" :key="music.path">
            <MusicFileEntry
                type="music"
                :name="music.name?.slice(0, music.name?.length - 5) || '_'"
                :path="music.path"
                @on:remove="loadCollection"
                @on:rename="loadCollection"
                v-show="music.name?.toLowerCase().includes(search_string.toLowerCase())"
            />
        </div>
    </div>
    <p v-else>
        Loading...
    </p>

    <Transition name="fade-in-fast">
        <PrimaryDialog
            :title="message().music_create_collection_button"
            v-model="createCollection.is_show"
            v-show="createCollection.is_show"
        >
            <PrimaryInput
                icon="tag"
                type="text"
                width="100%"
                :placeholder="message().music_create_collection_placeholder"
                v-model="createCollection.collection_name"
            />

            <PrimaryButton
                icon="add"
                @click="createCollection.Create"
            > {{ message().music_create_collection_button }} </PrimaryButton>
        </PrimaryDialog>
    </Transition>

    <ContextMenu
        v-if="section_el"
        :watch_element="section_el"
        v-model="is_open_context_menu"
    >
        <ContextMenuItem
            icon="create_new_folder"
            @click="createCollection.Open"
        > {{ message().music_create_collection_button }} </ContextMenuItem>

        <ContextMenuItem
            icon="audio_file"
            @click="importFromMIDI"
        > {{ message().music_import_midi_file_button }} </ContextMenuItem>
    </ContextMenu>
</section>
</template>

<style scoped lang="scss">
@import "../asset/scss/container.scss";

.container-md {
    min-height: calc(100vh - 37.78px);
}

.navbar {
    margin-bottom: 1rem;
    display: flex;
    align-items: center;
    margin-bottom: 1rem;

    .search {
        width: 100%;
        margin-right: 7px;
    }
}

.breadcrumb {
    margin: 1rem auto;

    div {
        display: inline;
    }
}

.list {
    position: relative;
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    grid-gap: .4rem;
}
</style>