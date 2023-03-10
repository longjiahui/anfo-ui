<template>
    <div
        ref="container"
        v-drop:[channel]
        @drop-drag-move="handleDragMove"
        @drop-drag-leave="handleDropDragLeave"
        @drop-drag-enter="handleDragEnter"
        @drop-drag-start="handleDragStart"
        @drop-drag-finish="_draggingKey = null"
        :class="['page orderable-container align-stretch', isHorizontal ? 'h' : 'v']">
        <transition-group @before-leave="handleBeforeLeave" :name="transitionName">
            <div v-for="(d, i) in middleware" :key="dataKey(d)"
                v-drag:[channel]="{ d, i }"
                @drag-finish="handleDragFinish"
                :class="[
                    'orderable-item',
                    isHorizontal ? 'height-full' : 'width-full',
                    getKey(d) === _draggingKey ? 'is-current':'']">
                <div :class="[
                    'item-content',
                    isHorizontal ? 'height-full' : 'width-full',
                ]">
                    <slot :data="d" :i="i"></slot>
                </div>
            </div>
        </transition-group>
    </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import utils from '@/scripts/utils'
import extend from 'extend'

let props = defineProps({
    datas: {
        type: Array,
        default: ()=>[]
    },
    dataKey: {
        type: Function,
        default: d=>d,
    },
    transitionName: {
        type: String,
        default: 'anfo-move-default',
    },
    channel: {
        type: String,
        default: 'anfo-orderable-container',
    },
    isHorizontal: {
        type: Boolean,
        default: false,
    },
    isOrderable: {
        type: Boolean,
        default: true,
    },
    isLeavable: {
        type: Boolean,
        default: false,
    },
    isEnterable: {
        type: Boolean,
        default: false,
    },
})
let emit = defineEmits(['change', 'update:datas'])

let container = ref(null)
let middleware = utils.createMiddleware(()=>{
    return extend(true, [], props.datas)
}, val=>emit('update:datas', val))

// ??????????????????????????????????????????????????????
// ???move????????????????????????????????? middleware???????????????
let _originalDragKeys = null
let _originalDragDatas = []
let _draggingKey = ref(null)
let _currentDragingIndex = -1
let _offsetRels = []
let _offsetRelKey = computed(()=>props.isHorizontal ? 'offsetLeft' : 'offsetTop')
let _heightRelKey = computed(()=>props.isHorizontal ? 'offsetWidth' : 'offsetHeight')

function handleDragFinish(e){
    emit('change', e.getValue('d'), _currentDragingIndex)
}

function getKey(data){
    return props.dataKey(data)
}
function getDataByKey(key){
    return middleware.value.find(d=>getKey(d) === key)
}

function handleDropDragLeave(e){
    if(props.isLeavable){
        let key = _draggingKey.value
        let ind = middleware.value.findIndex(d=>getKey(d) === key)
        middleware.value.splice(ind, 1)
        _currentDragingIndex = -1
    }
}

function getIndexByOffset(offset){
    let [x = -1, y = -1] = offset || []
    let rel = props.isHorizontal ? x : y
    let ind = _offsetRels.findLastIndex(val=>rel > val)
    return ind
}

function handleDragEnter(e){
    if(props.isEnterable){
        let offset = e.getData('offset')
        let { d, i } = e.getValue() || {}
        // ???????????????????????????value
        let key = getKey(d)
        if(!_originalDragDatas.some(i=>getKey(i) === key)){
            let index = getIndexByOffset(offset)
            _originalDragKeys.splice(index, 0, key)
            _originalDragDatas.splice(index, 0, d)
            _draggingKey.value = key
            // ???????????????????????? ??????????????????????????????????????? ???????????????????????????????????????
            let els = [...container.value.children]
            let last = els[_offsetRels.length - 1]
            _offsetRels.push(last[_offsetRelKey.value] + last[_heightRelKey.value])
            // console.debug(_offsetRels, last[_heightRelKey.value])
            // requestAnimationFrame(()=>_offsetRels.push(container.value.offsetHeight - last.offsetHeight))
            // requestAnimationFrame(()=>console.debug(last.getBoundingClientRect(), last.innerHTML, _heightRelKey.value, last, last[_offsetRelKey.value], last[_heightRelKey.value]))
        }
    }
}

function handleDragStart(e){
    // ?????????????????????????????????
    let { d, i } = e.getValue() || {}
    let key = getKey(d)
    if(middleware.value.some(i=>getKey(i) === key)){
        _draggingKey.value = key
    }
    _originalDragKeys = middleware.value.map(d=>getKey(d))
    _originalDragDatas = _originalDragKeys.map(getDataByKey)
    let els = [...container.value.children]
    _offsetRels = els.map(el=>el[_offsetRelKey.value])
}

function handleDragMove(e){
    let { offset = [] } = e.getData() || {}
    let ind = getIndexByOffset(offset)
    if(ind > -1 && _currentDragingIndex !== ind){
        _currentDragingIndex = ind
        let datas = [..._originalDragDatas]
        let draggingIndex = datas.findIndex(d=>getKey(d) === _draggingKey.value)
        if(draggingIndex > -1){
            if(props.isOrderable){
                datas.splice(ind, 0,
                    datas.splice(draggingIndex, 1)[0])
            }
            middleware.value = datas
        }
    }
}

function handleBeforeLeave(el){
    const {marginLeft, marginTop, width, height} = window.getComputedStyle(el)
    el.style.left = `${el.offsetLeft - parseFloat(marginLeft, 10)}px`
    el.style.top = `${el.offsetTop - parseFloat(marginTop, 10)}px`
    el.style.width = width
    el.style.height = height
}
</script>

<style lang="scss" scoped>
.orderable-item{
    // transition: border-radius .3s, background .3s;
    transition: all .3s ease;
    &.is-current{
        // opacity: 0;
        border-radius: 5px;
        background: rgba(0, 0, 0, .05);

        .item-content{
            opacity: 0;
        }
    }
    .item-content{
        transition: opacity .3s;
    }
}
</style>