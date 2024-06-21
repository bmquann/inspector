<template>
    <n-tree ref="treeInst" style="height: 90vh" virtual-scroll  :selected-keys="[props.selectedKey]"
        v-if="treeData" block-line :data="treeData" key-field="id" :render-label="renderLabel" :node-props="nodeProps"
        default-expand-all />
</template>

<script setup>
import { defineComponent, ref } from 'vue'


const showDropdown = ref(false)
const options = ref([])
const x = ref(0)
const y = ref(0)
const showIrrelevantNodes = ref(false)

const treeInst = ref()

const props = defineProps({
    data: Object,
    selectedKey: String | null
})


const emit = defineEmits(['hoverNode', 'clickNode', 'deHoverNode'])

const pattern = ref('123')
function sourceToTree(source) {
    if (!source) return []
    let n = {}
    n.id = source._id;
    n.text = source._type
    if (source.name) {
        n.text += " - " + source.name;
    }
    if (source.resourceId) {
        n.text += " - " + source.resourceId;
    }

    if (source.children) {
        n.children = []
        source.children.forEach(function (s) {
            n.children.push(sourceToTree(s))
        }.bind(this))
    }
    return n;
}
function renderLabel({ option }) {
    // const text = option?.text.split('-')[0]

    const listText = option.text?.split('-')
    if (!listText) return
    return h('div', {id: option.id}, [
        h('span', {style: {color: 'red'}}, listText[0]),

        h('span', {}, [
            h('i', {style: {color: 'blue'}},{default: () => listText[1] ? '- resourceId=' : ''}),
            h('span', {}, listText[1] ? `"${listText[1].trim()}"` : ''),
        ]),
    ])
}


function nodeProps({ option }) {
    return {
        onClick() {
            emit('clickNode', option.id)
        },
        onMouseover() {
            emit('hoverNode', option.id)
        },
        onMouseleave() {
            emit('deHoverNode', option.id)
        }
    };
}

const handleScrollToKey = (key) => {
    treeInst.value?.scrollTo({ key: props.selectedKey })
}

const treeData = computed(() => {
    return [sourceToTree(props.data)]
})

watch(() => props.selectedKey, (val) => {
    handleScrollToKey(val)
})
</script>
