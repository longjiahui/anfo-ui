<template>
    <div
        ref="container"
        :class="containerClass"
        :style="{
            '--layer': layer,
            ...containerStyle,
        }">
        <transition-group @before-leave="handleBeforeLeave" :name="transitionName" appear>
            <div
                v-for="(item, i) in middleware"
                :key="getKey(item) || i">
                <div class="h h-sem align-stretch">
                    <!-- <div v-if="getChildren(item).length > 0" @click="handleToggleSubLoops(item, i)" class="clickable h">
                        <CaretRightOutlined :class="['button-fold', folds[getKey(item)] ? 'is-fold':'']" />
                    </div> -->
                    <slot
                        v-bind="typeof item === 'object'?{
                        item,
                        datas: middleware,
                        i,
                        key: getKey(item,),
                        layer,
                        hasChildren: getChildren(item).length > 0,
                        prevHasChildren: middleware[i - 1] && getChildren(middleware[i - 1]).length > 0,
                        toggle: (_item = item, _i = i)=>handleToggleSubLoops(_item,  _i),
                        isFold: folds[getKey(item,)],
                        isLast: (isLast || !parent) && !(getChildren(item).length > 0) && i === middleware.length - 1,
                        isFirst: i === 0 && !parent,
                        parent,
                    } : { }"></slot>
                </div>
                <transition @after-enter="handleAfterEnter" name="fade-height" mode="out-in">
                    <div v-show="!folds[getKey(item)] && getChildren(item).length > 0"
                        class="sub-loop-wrapper">
                        <anfo-loop
                            v-bind="$props"
                            class="sub-loop"
                            :childrenKey="childrenKey"
                            v-model:datas="item[childrenKey]"
                            :layer="(+layer) + 1"

                            :parent="item"
                            :is-last="(isLast || !parent) && i === middleware.length - 1">
                            <template v-slot="item">
                                <slot v-bind="item"></slot>
                            </template>
                        </anfo-loop>
                    </div>
                </transition>
            </div>
        </transition-group>
    </div>
</template>
    
<script setup>
import { computed, nextTick, ref, watch } from 'vue'
import { CaretRightOutlined } from '@ant-design/icons-vue'
import utils from '@/scripts/utils'
import loopProps from './loopProps'

let props = defineProps({
    ...loopProps,
})

let container = ref(null)

let folds = ref({})

function handleAfterEnter(el){
    el.style.height = ''
    el.style.overflow = ''
}
async function handleToggleSubLoops(item , i){
    let itemRef = container.value.children[i]
    let currentFoldState = folds.value[getKey(item)]
    let toFoldState =!currentFoldState
    if(itemRef){
        let subLoopWrapper = [...itemRef.children].find(c=>[...c.classList].includes('sub-loop-wrapper'))
        if(subLoopWrapper){
            let subLoop = [...subLoopWrapper.children].find(c=>[...c.classList].includes('sub-loop'))
            let subLoopHeight
            if(toFoldState){
                subLoopWrapper.classList.add('overflow-hidden')
                subLoopHeight = subLoop.offsetHeight
            }else{
                subLoopWrapper.classList.remove('overflow-hidden')
                // ????????????el????????????fixed ?????????????????????????????????????????????????????????
                let el = subLoop.cloneNode(true)
                Object.assign(el.style, {
                    opacity: 0,
                    pointerEvents: 'none',
                    position: 'fixed',
                    top: 0,
                    left: 0,
                    height: 'auto',
                })
                document.body.appendChild(el)
                // ????????????UI????????????
                await new Promise(requestAnimationFrame)
                subLoopHeight = el.offsetHeight
                el.remove()
            }
            subLoopWrapper.style.height = subLoopHeight + 'px'
            await new Promise(r=>nextTick(r))
            folds.value[getKey(item)] = toFoldState
        }
    }
}

function getKey(item){
    return props.dataKey instanceof Function ? props.dataKey(item)
        : item?.[props.dataKey]
}
function getChildren(item){
    return item?.[props.childrenKey] || []
}

let emit = defineEmits(['update:datas'])
let middleware = utils.createMiddleware(()=>props.datas, async val=>{
    emit('update:datas', val)
})

function handleBeforeLeave(el){
    const {marginLeft, marginTop, width, height} = window.getComputedStyle(el)
    el.style.left = `${el.offsetLeft - parseFloat(marginLeft, 10)}px`
    el.style.top = `${el.offsetTop - parseFloat(marginTop, 10)}px`
    el.style.width = width
    el.style.height = height
}
</script>

<style lang="scss" scoped>
.fade-height-enter-active,
.fade-height-leave-active {
    transition: opacity .6s, height .3s;
}

.fade-height-enter-from,
.fade-height-leave-to {
    height: 0!important;
    opacity: 0;
}
.sub-loop-wrapper{
    // overflow: hidden;
}
.button-fold{
    position: relative;
    // position: absolute;
    // ??????plan-tree?????????
    left: -.15em;
    transition: transform .3s;
    transform: rotate(90deg);
    &.is-fold{
        transform: none;
    }
}
</style>