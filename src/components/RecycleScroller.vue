<template>
  <div
    v-observe-visibility="handleVisibilityChange"
    class="vue-recycle-scroller"
    :class="{
      ready,
      'page-mode': pageMode,
      [`direction-${direction}`]: true,
    }"
    @scroll.passive="handleScroll"
  >
    <div
      v-if="$slots.before"
      class="vue-recycle-scroller__slot"
    >
      <slot
        name="before"
      />
    </div>

    <div
      ref="wrapper"
      :style="{ [direction === 'vertical' ? 'minHeight' : 'minWidth']: totalSize + 'px' }"
      class="vue-recycle-scroller__item-wrapper"
    >
      <div
        v-for="view of pool"
        :key="view.nr.id"
        :style="ready ? { transform: `translate${direction === 'vertical' ? 'Y' : 'X'}(${view.position}px)`, zIndex: items.length - view.nr.index } : null"
        class="vue-recycle-scroller__item-view"
        @mouseenter="$event.target.classList.add('hover')"
        @mouseleave="$event.target.classList.remove('hover')"
      >
        <slot
          :item="view.item"
          :index="view.nr.index"
          :active="view.nr.used"
        />
      </div>
    </div>

    <div
      v-if="$slots.after"
      class="vue-recycle-scroller__slot"
    >
      <slot
        name="after"
      />
    </div>

    <ResizeObserver @notify="handleResize" />
  </div>
</template>

<script>
import { ObserveVisibility } from 'vue-observe-visibility'
import ScrollParent from 'scrollparent'
import config from '../config'
import { props, simpleArray } from './common'
import { supportsPassive } from '../utils'
import { ResizeObserver } from 'vue-resize'
let uid = 0

export default {
  name: 'RecycleScroller',

  components: {
    ResizeObserver,
  },

  directives: {
    ObserveVisibility,
  },

  props: {
    ...props,

    itemSize: {
      type: Number,
      default: null,
    },

    minItemSize: {
      type: [Number, String],
      default: null,
    },

    sizeField: {
      type: String,
      default: 'size',
    },

    typeField: {
      type: String,
      default: 'type',
    },

    buffer: {
      type: Number,
      default: 200,
    },

    pageMode: {
      type: Boolean,
      default: false,
    },

    prerender: {
      type: Number,
      default: 0,
    },

    emitUpdate: {
      type: Boolean,
      default: false,
    },
  },

  data () {
    return {
      pool: [],
      totalSize: 0,
      ready: false,
      hoverKey: null,
    }
  },

  computed: {
    sizes () {
      if (this.itemSize === null) {
        const sizes = {
          '-1': { accumulator: 0 },
        }
        const items = this.items
        const field = this.sizeField
        const minItemSize = this.minItemSize
        let computedMinSize = 10000
        let accumulator = 0
        let current
        for (let i = 0, l = items.length; i < l; i++) {
          current = items[i][field] || minItemSize
          if (current < computedMinSize) {
            computedMinSize = current
          }
          accumulator += current
          sizes[i] = { accumulator, size: current }
        }
        // eslint-disable-next-line
        this.$_computedMinItemSize = computedMinSize
        return sizes
      }
      return []
    },

    simpleArray,
  },

  watch: {
    items () {
      this.updateVisibleItems(true)
    },

    pageMode () {
      this.applyPageMode()
      this.updateVisibleItems(false)
    },

    sizes: {
      handler () {
        this.updateVisibleItems(false)
      },
      deep: true,
    },
  },

  created () {
    this.$_startIndex = 0
    this.$_endIndex = 0
    this.$_views = new Map()
    this.$_unusedViews = new Map()
    this.$_scrollAnimationRequest = null
    this.$_lastUpdateScrollPosition = 0

    // In SSR mode, we also prerender the same number of item for the first render
    // to avoir mismatch between server and client templates
    if (this.prerender) {
      this.$_prerender = true
      this.updateVisibleItems(false)
    }
  },

  mounted () {
    this.applyPageMode()
    const { totalSize } = this.startEndIndex(this.getScroll())
    this.$refs.wrapper.style[this.direction === 'vertical' ? 'minHeight' : 'minWidth'] = totalSize + 'px'
    this.$nextTick(() => {
      // In SSR mode, render the real number of visible items
      this.$_prerender = false
      this.updateVisibleItems(true)
      this.ready = true
    })
  },

  beforeDestroy () {
    this.removeListeners()
  },

  methods: {
    addView (pool, index, item, key, type) {
      const view = {
        item,
        position: 0,
      }
      const nonReactive = {
        id: uid++,
        index,
        used: true,
        key,
        type,
      }
      Object.defineProperty(view, 'nr', {
        configurable: false,
        value: nonReactive,
      })
      pool.push(view)
      return view
    },

    unuseView (view, fake = false) {
      const unusedViews = this.$_unusedViews
      const type = view.nr.type
      if (!unusedViews.get(type)) {
        unusedViews.set(type, [])
      }
      const unusedPool = unusedViews.get(type)
      unusedPool.push(view)
      if (!fake) {
        view.nr.used = false
        view.position = -9999
        this.$_views.delete(view.nr.key)
      }
    },

    handleResize () {
      this.$emit('resize')
      if (this.ready) this.updateVisibleItems(false)
    },

    handleScroll (event) {
      cancelAnimationFrame(this.$_scrollAnimationRequest)
      this.$_scrollAnimationRequest = requestAnimationFrame(() => {
        this.$_scrollAnimationRequest = false
        const { continuous } = this.updateVisibleItems(false, true)

        // It seems sometimes chrome doesn't fire scroll event :/
        // When non continous scrolling is ending, we force a refresh
        if (!continuous) {
          clearTimeout(this.$_refreshTimout)
          this.$_refreshTimout = setTimeout(this.handleScroll, 100)
        }
      })
    },

    handleVisibilityChange (isVisible, entry) {
      if (this.ready) {
        if (isVisible) {
          this.$emit('visible')
          requestAnimationFrame(() => {
            this.updateVisibleItems(false)
          })
        } else {
          this.$emit('hidden')
        }
      }
    },
    startEndIndex (scroll) {
      const itemSize = this.itemSize
      const items = this.items
      const count = items.length
      const sizes = this.sizes

      if (!count) {
        return {
          startIndex: 0,
          endIndex: 0,
          totalSize: 0,
        }
      }
      if (this.$_prerender) {
        return {
          startIndex: 0,
          endIndex: this.prerender,
          totalSize: null,
        }
      }

      this.$_lastUpdateScrollPosition = scroll.originalStart

      const buffer = this.buffer
      scroll.start -= buffer
      scroll.end += buffer

      // Variable size mode
      if (itemSize === null) {
        let h
        let a = 0
        let b = count - 1
        let i = ~~(count / 2)
        let oldI

        // Searching for startIndex
        do {
          oldI = i
          h = sizes[i].accumulator
          if (h < scroll.start) {
            a = i
          } else if (i < count - 1 && sizes[i + 1].accumulator > scroll.start) {
            b = i
          }
          i = ~~((a + b) / 2)
        } while (i !== oldI)
        i < 0 && (i = 0)
        let startIndex = i

        // For container style
        const totalSize = sizes[count - 1].accumulator

        // Searching for endIndex
        let endIndex
        for (endIndex = i; endIndex < count && sizes[endIndex].accumulator < scroll.end; endIndex++) {
          if (endIndex === -1) {
            endIndex = items.length - 1
          } else {
            endIndex++
            // Bounds
            endIndex > count && (endIndex = count)
          }
        }

        // I don't trust the sorcery above
        startIndex = Math.max(startIndex, 0)
        // Stop endIndex from being >items.length
        endIndex = Math.min(endIndex, count)

        return {
          startIndex,
          endIndex,
          totalSize,
        }
      }
      // Fixed size mode
      const startIndex = Math.max(0, ~~(scroll.start / itemSize))
      const endIndex = Math.min(count, Math.ceil(scroll.end / itemSize))
      const totalSize = count * itemSize

      return {
        startIndex,
        endIndex,
        totalSize,
      }
    },
    updateVisibleItems (checkItem, checkPositionDiff = false) {
      const itemSize = this.itemSize
      const minItemSize = this.$_computedMinItemSize
      const typeField = this.typeField
      const keyField = this.simpleArray ? null : this.keyField
      const items = this.items
      const count = items.length
      const sizes = this.sizes
      const views = this.$_views
      const unusedViews = this.$_unusedViews
      const pool = this.pool

      let scroll

      if (count && !this.$_prerender) {
        scroll = this.getScroll()

        // Skip update if use hasn't scrolled enough
        if (checkPositionDiff) {
          const positionDiff = Math.abs(scroll.originalStart - this.$_lastUpdateScrollPosition)

          if ((itemSize === null && positionDiff < minItemSize) || positionDiff < itemSize) {
            return {
              continuous: true,
            }
          }
        }
      }

      const { startIndex, endIndex, totalSize } = this.startEndIndex(scroll)

      if (endIndex - startIndex > config.itemsLimit) {
        this.itemsLimitError()
      }

      this.totalSize = totalSize

      let view

      const continuous = startIndex <= this.$_endIndex && endIndex >= this.$_startIndex

      if (this.$_continuous !== continuous) {
        if (continuous) {
          views.clear()
          unusedViews.clear()
          for (let i = 0, l = pool.length; i < l; i++) {
            view = pool[i]
            this.unuseView(view)
          }
        }
        this.$_continuous = continuous
      } else if (continuous) {
        for (let i = 0, l = pool.length; i < l; i++) {
          view = pool[i]
          if (view.nr.used) {
            // Update view item index
            if (checkItem) {
              view.nr.index = items.findIndex(
                item => keyField ? item[keyField] === view.item[keyField] : item === view.item,
              )
            }

            // Check if index is still in visible range
            if (
              view.nr.index === -1 ||
              view.nr.index < startIndex ||
              view.nr.index >= endIndex
            ) {
              this.unuseView(view)
            }
          }
        }
      }

      const unusedIndex = continuous ? null : new Map()

      let item, type, unusedPool
      let v
      for (let i = startIndex; i < endIndex; i++) {
        item = items[i]
        const key = keyField ? item[keyField] : item
        if (key == null) {
          throw new Error(`Key is ${key} on item (keyField is '${keyField}')`)
        }
        view = views.get(key)

        if (!itemSize && !sizes[i].size) {
          if (view) this.unuseView(view)
          continue
        }

        // No view assigned to item
        if (!view) {
          type = item[typeField]
          unusedPool = unusedViews.get(type)

          if (continuous) {
            // Reuse existing view
            if (unusedPool && unusedPool.length) {
              view = unusedPool.pop()
              view.item = item
              view.nr.used = true
              view.nr.index = i
              view.nr.key = key
              view.nr.type = type
            } else {
              view = this.addView(pool, i, item, key, type)
            }
          } else {
            // Use existing view
            // We don't care if they are already used
            // because we are not in continous scrolling
            v = unusedIndex.get(type) || 0

            if (!unusedPool || v >= unusedPool.length) {
              view = this.addView(pool, i, item, key, type)
              this.unuseView(view, true)
              unusedPool = unusedViews.get(type)
            }

            view = unusedPool[v]
            view.item = item
            view.nr.used = true
            view.nr.index = i
            view.nr.key = key
            view.nr.type = type
            unusedIndex.set(type, v + 1)
            v++
          }
          views.set(key, view)
        } else {
          view.nr.used = true
          view.item = item
        }

        // Update position
        if (itemSize === null) {
          view.position = sizes[i - 1].accumulator
        } else {
          view.position = i * itemSize
        }
      }

      this.$_startIndex = startIndex
      this.$_endIndex = endIndex

      if (this.emitUpdate) this.$emit('update', startIndex, endIndex)

      // After the user has finished scrolling
      // Sort views so text selection is correct

      return {
        continuous,
      }
    },

    getListenerTarget () {
      let target = ScrollParent(this.$el)
      // Fix global scroll target for Chrome and Safari
      if (window.document && (target === window.document.documentElement || target === window.document.body)) {
        target = window
      }
      return target
    },

    getScroll () {
      const { $el: el, direction } = this
      const isVertical = direction === 'vertical'

      if (this.pageMode) {
        const bounds = el.getBoundingClientRect()
        const boundsSize = isVertical ? bounds.height : bounds.width
        const originalStart = -(isVertical ? bounds.top : bounds.left)
        const originalSize = isVertical ? window.innerHeight : window.innerWidth
        let start = originalStart
        let size = originalSize

        if (start < 0) {
          size += start
          start = 0
        }
        if (start + size > boundsSize) {
          size = boundsSize - start
        }
        return {
          originalStart,
          start,
          end: start + size,
        }
      }

      if (isVertical) {
        return {
          originalStart: el.scrollTop,
          start: el.scrollTop,
          end: el.scrollTop + el.clientHeight,
        }
      } else {
        return {
          originalStart: el.scrollLeft,
          start: el.scrollLeft,
          end: el.scrollLeft + el.clientWidth,
        }
      }
    },

    applyPageMode () {
      if (this.pageMode) {
        this.addListeners()
      } else {
        this.removeListeners()
      }
    },

    addListeners () {
      this.listenerTarget = this.getListenerTarget()
      this.listenerTarget.addEventListener('scroll', this.handleScroll, supportsPassive ? {
        passive: true,
      } : false)
      this.listenerTarget.addEventListener('resize', this.handleResize)
    },

    removeListeners () {
      if (!this.listenerTarget) {
        return
      }

      this.listenerTarget.removeEventListener('scroll', this.handleScroll)
      this.listenerTarget.removeEventListener('resize', this.handleResize)

      this.listenerTarget = null
    },

    scrollToItem (index) {
      const { viewport, scrollDirection, scrollDistance } = this.scrollToPosition(index)
      viewport[scrollDirection] = scrollDistance
    },

    scrollToPosition (index) {
      const getPositionOfItem = (index) => {
        if (this.itemSize === null) {
          return index > 0 ? this.sizes[index - 1].accumulator : 0
        } else {
          return index * this.itemSize
        }
      }
      const position = getPositionOfItem(index)
      const direction = this.direction === 'vertical'
        ? { scroll: 'scrollTop', start: 'top' }
        : { scroll: 'scrollLeft', start: 'left' }

      if (this.pageMode) {
        const viewportEl = ScrollParent(this.$el)
        // HTML doesn't overflow like other elements
        const scrollTop = viewportEl.tagName === 'HTML' ? 0 : viewportEl[direction.scroll]
        const viewport = viewportEl.getBoundingClientRect()

        const scroller = this.$el.getBoundingClientRect()
        const scrollerPosition = scroller[direction.start] - viewport[direction.start]

        return {
          viewport: viewportEl,
          scrollDirection: direction.scroll,
          scrollDistance: position + scrollTop + scrollerPosition,
        }
      }

      return {
        viewport: this.$el,
        scrollDirection: direction.scroll,
        scrollDistance: position,
      }
    },

    itemsLimitError () {
      setTimeout(() => {
        console.log('It seems the scroller element isn\'t scrolling, so it tries to render all the items at once.', 'Scroller:', this.$el)
        console.log('Make sure the scroller has a fixed height (or width) and \'overflow-y\' (or \'overflow-x\') set to \'auto\' so it can scroll correctly and only render the items visible in the scroll viewport.')
      })
      throw new Error('Rendered items limit reached')
    },
  },
}
</script>

<style>
.vue-recycle-scroller {
  position: relative;
}

.vue-recycle-scroller.direction-vertical:not(.page-mode) {
  overflow-y: auto;
}

.vue-recycle-scroller.direction-horizontal:not(.page-mode) {
  overflow-x: auto;
}

.vue-recycle-scroller.direction-horizontal {
  display: flex;
}

.vue-recycle-scroller__slot {
  flex: auto 0 0;
}

.vue-recycle-scroller__item-wrapper {
  flex: 1;
  box-sizing: border-box;
  overflow: hidden;
  position: relative;
}

.vue-recycle-scroller.ready .vue-recycle-scroller__item-view {
  position: absolute;
  top: 0;
  left: 0;
  will-change: transform;
}

.vue-recycle-scroller.direction-vertical .vue-recycle-scroller__item-wrapper {
  width: 100%;
}

.vue-recycle-scroller.direction-horizontal .vue-recycle-scroller__item-wrapper {
  height: 100%;
}

.vue-recycle-scroller.ready.direction-vertical .vue-recycle-scroller__item-view {
  width: 100%;
}

.vue-recycle-scroller.ready.direction-horizontal .vue-recycle-scroller__item-view {
  height: 100%;
}
</style>
