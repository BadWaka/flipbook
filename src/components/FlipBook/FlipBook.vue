<template>
    <div
        ref="flipBook"
        class="flip-book"
        @touchstart="onTouchStart"
        @touchmove="onTouchMove"
    >
        <slot/>
    </div>
</template>

<script setup lang="ts">

import {
    ref,
    onMounted,
    defineEmits,
    watchEffect
} from 'vue';

// 定义 props
const props = defineProps({
    // 当前页索引
    selectedIndex: {
        type: Number,
        default: 0
    },
    // 自动播放
    auto: {
        type: Boolean,
        default: false
    },
    // 动效持续时间
    transitionDuration: {
        type: Number,
        default: 1000
    },
    // 预加载页数
    preLoad: {
        type: Number,
        default: 1
    },
    // 景深
    perspective: {
        type: String,
        default: '1000px'
    },
});

const emit = defineEmits([
    'sindexchange',
    'end',
    // 用户手动翻页
    'touchflip',
    'loadingImg'
]);

// 容器 DOM
const flipBook = ref(null);

// 所有页的 DOM
let itemDoms = null;

// 当前页的索引
// 组件内也要自己维护一个变量，因为如果是自动播放，要判断传入的 selectedIndex 是比当前的大还是小
let selectedIndex = props.selectedIndex;

watchEffect(() => {
    // 传入的 selectedIndex 是否比当前的大
    // 往右翻页
    if (props.selectedIndex > selectedIndex) {
        flipTo({
            containerDom: flipBook.value,
            doms: itemDoms,
            cIndex: selectedIndex,
            index: props.selectedIndex,
            direction: 'right',
            isTouchflip: false
        });
    }
    // 传入的 selectedIndex 是否比当前的小
    // 往左翻页
    else if (props.selectedIndex < selectedIndex) {
        flipTo({
            containerDom: flipBook.value,
            doms: itemDoms,
            cIndex: selectedIndex,
            index: props.selectedIndex,
            direction: 'left',
            isTouchflip: false
        });
    }
});

/**
 * 得到 DOM 的包裹 DOM
 * 可以得到左半边和右半边
 *
 * @param {*} originDom 原始 DOM
 * @param {*} direction 方向
 */
function getWrapperDom(originDom, options) {
    const width = options.width;
    const height = options.height;
    const direction = options.direction;
    const name = options.name;
    const isReverse = options.isReverse;

    const cloneDom = originDom.cloneNode(true);
    cloneDom.style.display = 'block';
    cloneDom.style.position = 'absolute';
    cloneDom.style.width = `${width}px`;
    cloneDom.style.height = `${height}px`;
    if (direction === 'right') {
        cloneDom.style.marginLeft = `-${width / 2}px`;
    }

    const wrapperDom = document.createElement('div');
    wrapperDom.classList.add(`${name}`);
    wrapperDom.classList.add(`${direction}`);
    wrapperDom.style.position = 'absolute';
    wrapperDom.style.top = '0';
    wrapperDom.style.left = '0';
    wrapperDom.style.width = `${width / 2}px`;
    wrapperDom.style.height = `${height}px`;
    wrapperDom.style.overflow = 'hidden';
    // 导致 iOS 有问题的罪魁祸首
    // wrapperDom.style.transform = 'translateZ(1px)';

    const transformStyle = 'preserve-3d';
    wrapperDom.style.webkitTransformStyle = transformStyle;
    wrapperDom.style.transformStyle = transformStyle;

    const backfaceVisibility = 'hidden';
    wrapperDom.style.webkitBackfaceVisibility = backfaceVisibility;
    wrapperDom.style.backfaceVisibility = backfaceVisibility;

    if (isReverse) {
        // wrapperDom.style.transform = 'scale(-1, 1)';
        const transform = 'rotateY(-180deg)';
        wrapperDom.style.webkitTransform = transform;
        wrapperDom.style.transform = transform;
    }

    wrapperDom.appendChild(cloneDom);

    return wrapperDom;
}

// 是否正在翻页
let isFlipping = false;

// 加载的图片状态
const loadImageStatus = ref({});

// 加载图片
function loadImage(url) {
    return new Promise((resolve, reject) => {
        const img = new Image();
        img.onload = () => resolve(url);
        img.onerror = () => reject(new Error(`Load error for ${url}`));
        img.src = url;
    });
}

/**
 * 翻到某一页
 *
 * @param {number} cIndex 当前页的索引
 * @param {number} index 目标页的索引
 * @param {string} direction 翻页方向；left：向左；right：向右
 */
function flipTo(options) {

    // 如果正在翻页，直接返回
    if (isFlipping) {
        return;
    }

    // 容器 DOM
    const containerDom = options.containerDom || flipBook.value;
    // 页面 DOM 数组
    const doms = options.doms || itemDoms.value;
    const cIndex = options.cIndex;
    const index = options.index;
    const direction = options.direction || 'right';
    const isTouchflip = options.isTouchflip;

    // 当前页的 DOM
    const cDom = doms[cIndex];
    // 目标页的 DOM
    const tDom = doms[index];

    // 超出页数
    if (index > doms.length - 1) {
        emit('end');
        return;
    }

    // 目标页的 DOM 不存在，直接返回
    if (!tDom) {
        return;
    }

    // 页数不变，也不返回
    if (cIndex === index) {
        return;
    }

    // 获取当前页的背景图片
    const style = getComputedStyle(tDom);
    const backgroundImageUrl = style.backgroundImage.slice(5, -2).replace(/["']/g, '');

    // 没加载过或者加载失败，重新加载
    if (!loadImageStatus.value[backgroundImageUrl] || loadImageStatus.value[backgroundImageUrl] === 'error') {
        loadPreDoms([tDom]);
        emit('loadingImg', {index, isTouchflip});
        return;
    }
    else if (loadImageStatus.value[backgroundImageUrl] === 'loading') {
        // 加载loading中 暂时返回
        emit('loadingImg', {index, isTouchflip});
        return;
    }

    // 不能放在 return 前面啊，否则会出问题
    isFlipping = true;

    // 记录宽高
    const width = cDom.offsetWidth;
    const height = cDom.offsetHeight;

    /**
     * 用两个「半页」拼成「一整页」
     */

    /**
     * 半页（老）
     */

    const halfPageOld = getWrapperDom(cDom, {
        width,
        height,
        direction: direction === 'right'
            ? 'left'
            : 'right',
        name: 'halfPageOld'
    });
    // 如果是向左翻页，半页（老）的 left 需要设置为宽度的一半
    if (direction === 'left') {
        halfPageOld.style.left = `${width / 2}px`;
    }
    // 不加会被目标页挡住
    halfPageOld.style.zIndex = 1;
    // 插入到容器 DOM 中，在当前页 DOM 之前
    containerDom.insertBefore(halfPageOld, cDom);

    /**
     * 半页（新）
     */

    // 它有正反两页
    // 正面是当前页的 DOM 的右（翻页方向默认为右）侧
    // 反面是目标页 DOM 的左（翻页方向默认为右）侧
    const halfPageNew = document.createElement('div');
    halfPageNew.classList.add('halfPageNew');
    halfPageNew.style.position = 'absolute';
    halfPageNew.style.top = '0';

    // halfPageNew.style.left = direction === 'right'
    //     ? `${width / 2}px`
    //     : 0;
    halfPageNew.style.left = `${width / 2}px`;
    halfPageNew.style.width = `${width / 2}px`;
    halfPageNew.style.height = `${height}px`;

    const transition = `transform ${props.transitionDuration}ms linear`;
    halfPageNew.style.webkitTransition = transition;
    halfPageNew.style.transition = transition;

    // 如果是往左滑，需要把 rotateY 置为 -180，避免在 iOS 15plus 上出现翻页方向不对的问题
    // const transform = `perspective(${props.perspective}) rotateY(${direction === 'right' ? 0 : -180}deg)`;
    const transform = `rotateY(${direction === 'right' ? 0 : -180}deg) translateX(-0.5px)`;
    halfPageNew.style.webkitTransform = transform;
    halfPageNew.style.transform = transform;

    // const transformOrigin = `${direction === 'right' ? 'left' : 'right'} center`;
    const transformOrigin = 'left center';
    halfPageNew.style.webkitTransformOrigin = transformOrigin;
    halfPageNew.style.transformOrigin = transformOrigin;

    const transformStyle = 'preserve-3d';
    halfPageNew.style.webkitTransformStyle = transformStyle;
    halfPageNew.style.transformStyle = transformStyle;

    const backfaceVisibility = 'visible';
    halfPageNew.style.webkitBackfaceVisibility = backfaceVisibility;
    halfPageNew.style.backfaceVisibility = backfaceVisibility;

    halfPageNew.style.willChange = 'transform';

    // 不加会被目标页挡住
    halfPageNew.style.zIndex = 2;

    // 当前页 DOM 的翻页方向侧
    const cDomDirection = getWrapperDom(cDom, {
        width,
        height,
        direction,
        name: 'cDomDirection',
        // 左滑时翻转
        isReverse: direction === 'left'
    });

    // 目标页 DOM 的翻页反方向侧
    tDom.style.display = 'block';
    tDom.style.opacity = 1;

    const tDomReverseDirection = getWrapperDom(tDom, {
        width,
        height,
        direction: direction === 'right'
            ? 'left'
            : 'right',
        name: 'tDomReverseDirection',
        // 右滑时翻转
        isReverse: direction === 'right'
    });

    // 都放进半页（新）
    halfPageNew.appendChild(cDomDirection);
    halfPageNew.appendChild(tDomReverseDirection);

    // 把右半页插入父容器
    containerDom.insertBefore(halfPageNew, cDom);

    /**
     * 将原始 DOM 隐藏
     */
    cDom.style.display = 'block';
    cDom.style.opacity = 0;

    // 新预加载页doms
    const newPreDoms = [];

    // 更新预加载页
    doms.forEach((item, i) => {
        if (Math.abs(index - i) <= props.preLoad && i !== index) {
            item.style.display = 'block';
            item.style.opacity = 0;
            newPreDoms.push(item);
        }
    });

    loadPreDoms(newPreDoms);

    /**
     * 执行翻转动画
     */
    setTimeout(() => {
        // 得等 DOM 加载到页面中完成，再设置样式才能生效
        // translateX(${direction === 'right' ? -.1 : .1}px)
        // const transform = `perspective(${props.perspective}) rotateY(${direction === 'right' ? -180 : 0}deg)`;
        const transform = `rotateY(${direction === 'right' ? -180 : 0}deg) translateX(-0.5px)`;
        halfPageNew.style.webkitTransform = transform;
        halfPageNew.style.transform = transform;
    }, 0);

    // PC 上滑动快的话，transitionend 会出现未被调用的问题
    setTimeout(() => {

        // 把新增用来做动画的 DOM 删掉
        containerDom.removeChild(halfPageOld);
        containerDom.removeChild(halfPageNew);

        // 把当前页索引设置为目标页的索引
        selectedIndex = index;
        // 触发 sindexchange 事件
        emit('sindexchange', index);

        // 结束正在翻页状态
        isFlipping = false;

    }, props.transitionDuration);

    // 动画结束事件
    // halfPageNew.addEventListener('transitionend', () => {

    //     // 把新增用来做动画的 DOM 删掉
    //     containerDom.removeChild(halfPageOld);
    //     containerDom.removeChild(halfPageNew);

    //     // 把当前页索引设置为目标页的索引
    //     selectedIndex = index;
    //     // 触发 sindexchange 事件
    //     emit('sindexchange', index);

    //     // 结束正在翻页状态
    //     isFlipping = false;

    // });

}

// TODO: 根据起点终点返回方向 1向上 2向下 3向左 4向右 0未滑动
function getDirection(angx, angy) {
    var result = 0;

    // 如果滑动距离太短
    if (Math.abs(angx) < 2 && Math.abs(angy) < 2) {
        return result;
    }

    var angle = getAngle(angx, angy);
    if (angle >= -135 && angle <= -45) {
        result = 1;
    } else if (angle > 45 && angle < 135) {
        result = 2;
    } else if ((angle >= 135 && angle <= 180) || (angle >= -180 && angle < -135)) {
        result = 3;
    } else if (angle >= -45 && angle <= 45) {
        result = 4;
    }

    return result;
}

// 获得角度
function getAngle(angx, angy) {
    return Math.atan2(angy, angx) * 180 / Math.PI;
}

// 加载doms中的图片
function loadPreDoms(doms) {
    doms.forEach(item => {
        // 获取图片的背景图
        const style = getComputedStyle(item);
        const backgroundImageUrl = style.backgroundImage.slice(5, -2).replace(/["']/g, '');
        const itemStatus = loadImageStatus.value[backgroundImageUrl];

        // 图片加载中 或者 已经加载成功，直接跳过
        if (itemStatus === 'loading' || itemStatus === 'success') {
            return;
        }

        // 设置加载状态为 loading
        loadImageStatus.value[backgroundImageUrl] = 'loading';

        loadImage(backgroundImageUrl).then(() => {
            // 设置加载状态为 success
            loadImageStatus.value[backgroundImageUrl] = 'success';
        }).catch(error => {
            // 设置加载状态为 error
            loadImageStatus.value[backgroundImageUrl] = 'error';
        });
    });
}

// 预加载图片doms
const preLoadDoms = [];

onMounted(async () => {

    // 拿到 flip-book-item DOMs
    itemDoms = document.querySelectorAll('.flip-book-item');

    // 隐藏其他页
    itemDoms.forEach((item, index) => {
        if (index !== props.selectedIndex) {
            // 预加载页
            if (Math.abs(props.selectedIndex - index) <= props.preLoad) {
                item.style.display = 'block';
                item.style.opacity = 0;
                // 预加载的页，添加到 preLoadDoms 中 + 当前页
                preLoadDoms.push(item, itemDoms[props.selectedIndex]);
            } else {
                // 预加载之外的页，隐藏
                item.style.display = 'none';
            }
        }
    });

    // 预加载dom中的图片
    loadPreDoms(preLoadDoms);

});

let startX = 0;
let startY = 0;
// 触发翻页次数；为了解决持续滑动一直翻页的问题，新增这一参数进行控制；一次 touchstart - touchend 事件之间最多只触发一次翻页
let emitFlipCount = 0;

function onTouchStart(e) {
    startX = e.touches[0].clientX;
    startY = e.touches[0].clientY;
    emitFlipCount = 0;
}

function onTouchMove(e) {
    let currentX = e.touches[0].clientX; // 当前触摸点的X坐标
    let currentY = e.touches[0].clientY; // 当前触摸点的Y坐标

    // 计算当前触摸点与初始触摸点的差值，得到滑动的距离
    let diffX = currentX - startX;
    let diffY = currentY - startY;

    // 获得方向
    let direction = getDirection(diffX, diffY);

    // diffX < 0 向左滑 = 向右翻页
    if (direction === 3 && Math.abs(diffX) > 10 && emitFlipCount < 1) {
        nextClick();
        emit('touchflip', {
            direction: 'right',
            cIndex: props.selectedIndex,
            index: props.selectedIndex + 1
        });
        emitFlipCount++;
    }

    // diffX > 0 向右滑 = 向左翻页
    else if (direction === 4 && Math.abs(diffX) > 10 && emitFlipCount < 1) {
        lastClick();
        emit('touchflip', {
            direction: 'left',
            cIndex: props.selectedIndex,
            index: props.selectedIndex - 1
        });
        emitFlipCount++;
    }
}

function lastClick() {
    flipTo({
        containerDom: flipBook.value,
        doms: itemDoms,
        cIndex: props.selectedIndex,
        index: props.selectedIndex - 1,
        direction: 'left',
        isTouchflip: true
    });
}

function nextClick() {
    flipTo({
        containerDom: flipBook.value,
        doms: itemDoms,
        cIndex: props.selectedIndex,
        index: props.selectedIndex + 1,
        direction: 'right',
        isTouchflip: true
    });
}

function firstClick() {
    flipTo({
        containerDom: flipBook.value,
        doms: itemDoms,
        cIndex: props.selectedIndex,
        index: 0,
        direction: 'left'
    });
}

function endClick() {
    flipTo({
        containerDom: flipBook.value,
        doms: itemDoms,
        cIndex: props.selectedIndex,
        index: itemDoms.length - 1,
        direction: 'right'
    });
}

</script>

<style lang="scss" scoped>
.flip-book {
    position: relative;
    height: 100%;
    overflow: hidden;
    perspective: 1000px;
}
</style>
