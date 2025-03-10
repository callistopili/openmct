<template>
<div
    class="l-pane"
    :class="{
        'l-pane--horizontal-handle-before': type === 'horizontal' && handle === 'before',
        'l-pane--horizontal-handle-after': type === 'horizontal' && handle === 'after',
        'l-pane--vertical-handle-before': type === 'vertical' && handle === 'before',
        'l-pane--vertical-handle-after': type === 'vertical' && handle === 'after',
        'l-pane--collapsed': collapsed,
        'l-pane--reacts': !handle,
        'l-pane--resizing': resizing === true
    }"
>
    <div
        v-if="handle"
        class="l-pane__handle"
        @mousedown="start"
    ></div>
    <div class="l-pane__header">
        <span v-if="label"
              class="l-pane__label"
        >{{ label }}</span>
        <slot name="controls"></slot>
        <button
            v-if="collapsable"
            class="l-pane__collapse-button c-icon-button"
            @click="toggleCollapse"
        ></button>
    </div>
    <button
        class="l-pane__expand-button"
        @click="toggleCollapse"
    >
        <span class="l-pane__expand-button__label">{{ label }}</span>
    </button>
    <div class="l-pane__contents">
        <slot></slot>
    </div>
</div>
</template>

<script>
const COLLAPSE_THRESHOLD_PX = 40;
const HIDE_TREE_PARAM = 'hideTree';
const HIDE_INSPECTOR_PARAM = 'hideInspector';
const PANE_INSPECTOR = 'Inspect';
const PANE_TREE = 'Browse';

export default {
    inject: ['openmct'],
    props: {
        handle: {
            type: String,
            default: '',
            validator: function (value) {
                return ['', 'before', 'after'].indexOf(value) !== -1;
            }
        },
        collapsable: {
            type: Boolean,
            default: false
        },
        label: {
            type: String,
            default: ''
        }
    },
    data() {
        return {
            collapsed: false,
            resizing: false
        };
    },
    beforeMount() {
        this.type = this.$parent.type;
        this.styleProp = (this.type === 'horizontal') ? 'width' : 'height';
    },
    async mounted() {
        await this.$nextTick();
        // Hide tree and/or inspector pane if specified in URL
        this.handleHideUrl();
        this.openmct.router.on('change:params', this.handleHideUrl);
    },
    beforeDestroy() {
        this.openmct.router.off('change:params', this.handleHideUrl);
    },
    methods: {
        toggleCollapse: function (e) {
            let target = this.label === PANE_TREE ? HIDE_TREE_PARAM : HIDE_INSPECTOR_PARAM;
            this.collapsed = !this.collapsed;
            if (this.collapsed) {
                this.handleCollapse();
                this.addHideParam(target);
            } else {
                this.handleExpand();
                this.removeHideParam(target);
            }
        },
        handleHideUrl: function () {
            if (!this.collapsable) {
                return;
            }

            let hideTreeParam = this.openmct.router.getSearchParam(HIDE_TREE_PARAM);
            let hideInspectorParam = this.openmct.router.getSearchParam(HIDE_INSPECTOR_PARAM);
            let hideTree = hideTreeParam === 'true' && this.label === PANE_TREE;
            let hideInspector = hideInspectorParam === 'true' && this.label === PANE_INSPECTOR;
            if (hideTree || hideInspector) {
                this.collapsed = true;
                this.handleCollapse();
            } else {
                this.collapsed = false;
                this.handleExpand();
            }
        },
        addHideParam: function (target) {
            this.openmct.router.setSearchParam(target, 'true');
        },
        removeHideParam: function (target) {
            this.openmct.router.deleteSearchParam(target);
        },
        handleCollapse: function () {
            this.currentSize = (this.dragCollapse === true) ? this.initial : this.$el.style[this.styleProp];
            this.$el.style[this.styleProp] = '';
        },
        handleExpand: function () {
            this.$el.style[this.styleProp] = this.currentSize;
            delete this.currentSize;
            delete this.dragCollapse;
        },
        trackSize: function () {
            if (!this.dragCollapse === true) {
                if (this.type === 'vertical') {
                    this.initial = this.$el.offsetHeight;
                } else if (this.type === 'horizontal') {
                    this.initial = this.$el.offsetWidth;
                }
            }
        },
        getPosition: function (event) {
            return this.type === 'horizontal'
                ? event.pageX
                : event.pageY;
        },
        getNewSize: function (event) {
            let delta = this.startPosition - this.getPosition(event);
            if (this.handle === "before") {
                return `${this.initial + delta}px`;
            }

            if (this.handle === "after") {
                return `${this.initial - delta}px`;
            }
        },
        updatePosition: function (event) {
            let size = this.getNewSize(event);
            let intSize = parseInt(size.substr(0, size.length - 2), 10);
            if (intSize < COLLAPSE_THRESHOLD_PX && this.collapsable === true) {
                this.dragCollapse = true;
                this.end();
                this.toggleCollapse();
            } else {
                this.$el.style[this.styleProp] = size;
            }
        },
        start: function (event) {
            event.preventDefault(); // stop from firing drag event

            this.startPosition = this.getPosition(event);
            document.body.addEventListener('mousemove', this.updatePosition);
            document.body.addEventListener('mouseup', this.end);
            this.resizing = true;
            this.$emit('start-resizing');
            this.trackSize();
        },
        end: function (event) {
            document.body.removeEventListener('mousemove', this.updatePosition);
            document.body.removeEventListener('mouseup', this.end);
            this.resizing = false;
            this.$emit('end-resizing');
            this.trackSize();
        }
    }
};
</script>
