<script setup lang="ts">
import { computed, ref, type Ref } from 'vue';

type UTModList = {
    file_name: string,
    list: Ref<UTMod[]>
}
type UTMod = {
    id: string,
    name: string,
    is_conflicting: boolean
}
const data_unturned_mods: Ref<UTModList[]> = ref([])
const identicals: Ref<string[]> = ref([])
const has_some_identical = computed(()=> identicals.value.length > 0)
const ref_hint_id = ref("")

function on_dragover(event: DragEvent){
    event.preventDefault()
}
function on_drop(event: DragEvent){
    if (! event.dataTransfer) return
    const files = [...event.dataTransfer.items].filter((item) => item.kind === "file")
    
    if (files.length === 0) return
    event.preventDefault()
    const valid_files = files.filter(x => x.type === "text/plain")
    if (valid_files.length === 0) return
    let wait_for_all_parsing = valid_files.length
    data_unturned_mods.value = []
    valid_files.forEach((raw_file, file_index)=>{
        const file = raw_file.getAsFile()
        if (!file) return
        data_unturned_mods.value.push({
            file_name: file?.name || "unknown name",
            list: ref([])
        } satisfies UTModList)

        file.text().then((data: string)=>{
            text_data_to_parsed(data, file_index)
            wait_for_all_parsing -= 1
            if (wait_for_all_parsing === 0){
                look_for_identicals()
            }
        })
    })
}
const regex_id = /^\d+/
function text_data_to_parsed(data: string, panel_id: number){
    const target_data_ref = data_unturned_mods.value[panel_id]
    if (!target_data_ref) return
    const list = data.trim().split('\n').map(x => {
        return {
            id: x.match(regex_id)?.[0] || "",
            name: x.replace(regex_id, ''),
            is_conflicting: false
        } satisfies UTMod
    })
    target_data_ref.list.value = list
}


function look_for_identicals(){
    compare_n_list_generic(data_unturned_mods.value.map(x => x.list.value))

}
function compare_n_list_generic(list: UTMod[][]){
    if (list.length <= 1) return
    const first  = list[0] as UTMod[]
    const second = list[1] as UTMod[]
    const rest   = list.slice(2)
    compare_list_two(first, second)
    compare_n_list_generic([first, ...rest])
    compare_n_list_generic([second, ...rest])
}

function compare_list_two(list_1: UTMod[], list_2: UTMod[]){
     list_1.forEach(({id: id_1})=>{
        list_2.forEach(({id: id_2})=>{
            if (id_1 === id_2) {
                identicals.value.push(id_1)
            }
        })
    })
}

function hint_hover(id: string){
    ref_hint_id.value = id
}
function hint_leave(){
    ref_hint_id.value = ""
}
</script>
<template>
<main @drop="on_drop" @dragover="on_dragover">
    <div v-if="data_unturned_mods.length === 0">
        <h1>
            Yep, you can drag and drop your files here now, 
        </h1>
        <p>multiple files at once</p>
    </div>
    <div class="panel" v-for="modlist in data_unturned_mods">
        <legend> File: {{ modlist.file_name }}</legend>
        <div class="text-box">
            <template v-for="{name, id} of modlist.list.value">
            <div class="mod-id-row" v-if="(has_some_identical && identicals.find(x => x === id)) || !has_some_identical"
                @mouseover="hint_hover(id)" @mouseleave="hint_leave" :class="id === ref_hint_id ? 'hint' : ''">
                <span class="mod-id-id" :class="has_some_identical ? 'mod-id-id-wrong' : ''"> {{ id }} </span>
                <span> {{ name }} </span>
            </div>
            </template>
            
        </div>
    </div>
</main>
</template>
<style scoped>
main{
    width: 100%;
    height: 100%;
    display: flex;
}
.panel{
    flex-grow: 1;
    display: flex;
    flex-direction: column;
    max-width: 50%;
    padding: 0.5em;
}
legend{
    margin-left: auto;
    margin-right: auto;
}
.text-box{
    flex-grow: 1;
    padding: 0.5em;
}
.mod-id-row{
    display: flex;
}
.mod-id-id{
    color: green;
}
.mod-id-id-wrong{
    color: red;
}
.hint{
    background-color: orange;
    color: purple;
}
</style>
