
<template>
    <div
        class="wrapper"
    >
        <FlipBook
            :selected-index="0"
        >
            <div
                v-for="item in list"
                :style="{
                    backgroundImage: `url(${item.src})`
                }"
                class="item flip-book-item"
            >
            </div>
        </FlipBook>
    </div>
</template>

<script setup lang="ts">
import {
    ref
} from 'vue';
import FlipBook from './components/FlipBook/FlipBook.vue';

const list = ref([
    {
        src: 'https://5b0988e595225.cdn.sohucs.com/images/20200105/d65d774079e242f19429a8bb730394f1.jpeg'
    },
    {
        src: 'https://img0.baidu.com/it/u=3549641921,2392395869&fm=253&fmt=auto&app=138&f=JPEG?w=500&h=890'
    },
    {
        src: 'https://img2.baidu.com/it/u=607246426,4214908815&fm=253&fmt=auto&app=138&f=JPEG?w=800&h=1281'
    }
]);
</script>

<style scoped lang="scss">
.wrapper {
    height: 100vh;
}
.item {
    position: absolute;
    top: -1px;
    left: -1px;
    right: -1px;
    bottom: -1px;
    background-size: cover;
    background-repeat: no-repeat;
}
</style>
