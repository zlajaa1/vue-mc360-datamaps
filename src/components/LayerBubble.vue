<template>
    <g class="bubbles">
        <circle :ref="`${name}`" v-for="(item, index) in bubblesData" :key="index"
            :class="name"
            :cx="latLng(item)[0]"
            :cy="latLng(item)[1]"
            :r="radius(item)"
            :style="styleAttributes[index]"
            style="cursor:pointer;"
            @click="handleClickCallback($event, item, index)"
            @mouseover="handleMouseOver($event, item, index)"
            @mouseout="handleMouseOut(index)"
        >
        <animate attributeName="r" begin="0s" dur="400ms" from="0" :to="radius(item)"></animate>
        </circle>
        <text
            :ref="`${name}-text`"
            v-for="(item, index) in bubblesData"
            :key="`${index}-text`"
            fill="white"
            :x="latLng(item)[0]"
            :y="latLng(item)[1]">
            {{ item.bubbleText }}
        </text>
        <rect
            v-if="item.deviationText != 0"
            :ref="`${name}-rectBox`"
            v-for="(item, index) in bubblesData"
            :key="`${index}-pill`"
            :x="latLng(item)[0]"
            :y="latLng(item)[1]"
            width="38" height="14" fill="black" ry="7" rx="7"/>
        <circle
            v-if="item.deviationText != 0"
            :ref="`${name}-mark`"
            v-for="(item, index) in bubblesData"
            :key="`${index}-mark`"
            :class="name"
            fill="red"
            :cx="latLng(item)[0]"
            :cy="latLng(item)[1]"
            :r="5"
        ></circle>
        <text
            v-if="item.deviationText != 0"
            :ref="`${name}-text-pill`"
            v-for="(item, index) in bubblesData"
            :key="`${index}-text-pill`"
            fill="white"
            font-size="12px"
            :x="latLng(item)[0]"
            :y="latLng(item)[1]">
            {{ item.deviationText }}
        </text>
    </g>
</template>

<script>
import { val } from './helper'
export default {
    name: 'layer-bubble',
    props: ['bubblesConfig', 'path', 'projection', 'data'],
    data () {
        return {
            name: 'datamaps-bubble',
            styleAttributes: {},
            previousAttributes: {}
        }
    },
    computed: {
        bubbles () {
            return this.$refs[`${this.name}`]
        },
        options () {
            return this.bubblesConfig
        },
        bubblesData () {
            return this.options.data
        },
    },
    mounted() {
        this.$nextTick(() => {
            this.setPosition();
        });
    },
    methods: {
        setPosition () {
            this.bubbles.forEach((bubbleParent, index) => {
                let parentRect = bubbleParent.getBBox();
                // Main text centering
                let mainText = this.$refs[`${this.name}-text`][index];
                mainText.setAttribute('x', parentRect.x + (parentRect.width / 2) - (mainText.getBBox().width / 2));
                mainText.setAttribute('y', parentRect.y + (parentRect.height / 2) + 5 );
                // Rect box centering
                if(typeof this.$refs[`${this.name}-rectBox`][index] == 'undefined') return;
                let rectBox = this.$refs[`${this.name}-rectBox`][index];
                rectBox.setAttribute('x', parentRect.x + parentRect.width - (parentRect.width / 3));
                rectBox.setAttribute('y', parentRect.y + 5);
                // Indicator centering
                let mark = this.$refs[`${this.name}-mark`][index];
                mark.setAttribute('cx', rectBox.getBBox().x + 8);
                mark.setAttribute('cy', rectBox.getBBox().y + 7);
                // Pill text centering
                let pillText = this.$refs[`${this.name}-text-pill`][index];
                pillText.setAttribute('x', rectBox.getBBox().x + 15);
                pillText.setAttribute('y', rectBox.getBBox().y + 11);
            })
        },
        styles (datum, index) {
            const data = {
                stroke: val(datum.borderColor, this.options.borderColor, datum),
                strokeWidth: val(datum.borderWidth, this.options.borderWidth, datum),
                strokeOpacity: val(datum.borderOpacity, this.options.highlightBorderOpacity, datum),
                fill: this.options.fills[val(datum.fillKey, this.options.fillKey, datum)] || this.options.fills.defaultFill,
                fillOpacity: val(datum.fillOpacity, this.options.highlightFillOpacity, datum)
            }
            this.$set(this.styleAttributes, index, data)
        },
        radius (datum) {
            return val(datum.radius, this.options.radius, datum)
        },
        datumHasCoords (datum) {
            return typeof datum !== 'undefined' && typeof datum.latitude !== 'undefined' && typeof datum.longitude !== 'undefined'
        },
        latLngToXY (lat, lng) {
            return this.projection([lng, lat])
        },
        latLng (datum) {
            if (datum.region && this.data[datum.region]) {
                return this.latLngToXY(this.data[datum.region].coordinates.latitude, this.data[datum.region].coordinates.longitude)
            } else if (datum.region && !this.data[datum.region]) {
                return this.latLngToXY(datum.coordinates.latitude, datum.coordinates.longitude)
            } else if (this.datumHasCoords(datum)) {
                return this.latLngToXY(datum.latitude, datum.longitude)
            } else if (datum.centered === 'USA') {
                return this.projection([-98.58333, 39.83333])
            } else {
                return this.path.centroid(this.data[datum.centered])
            }
        },
        handleMouseOver (event, datum, index) {
            const target = event.target
            const previousAttributes = {
                fill: target.style.fill,
                stroke: target.style.stroke,
                strokeWidth: target.style.strokeWidth,
                fillOpacity: target.style.fillOpacity
            }
            this.$set(this.previousAttributes, index, previousAttributes)
            const { highlightOnHover, popupOnHover, highlightFillColor, highlightBorderColor, highlightBorderWidth, highlightBorderOpacity, highlightFillOpacity } = this.options
            if (highlightOnHover || popupOnHover) {
                const data = {
                    fill: val(datum.highlightFillColor, highlightFillColor, datum),
                    stroke: val(datum.highlightBorderColor, highlightBorderColor, datum),
                    strokeWidth: val(datum.highlightBorderWidth, highlightBorderWidth, datum),
                    strokeOpacity: val(datum.highlightBorderOpacity, highlightBorderOpacity, datum),
                    fillOpacity: val(datum.highlightFillOpacity, highlightFillOpacity, datum)
                }
                this.$set(this.styleAttributes, index, data)
            }
            if (popupOnHover) this.$emit('show:popup', { event, datum })
        },
        handleMouseOut (index) {
            const { highlightOnHover, popupOnHover } = this.options
            if (highlightOnHover) {
                const data = this.previousAttributes[index]
                this.$set(this.styleAttributes, index, data)
            }
            if (popupOnHover) this.$emit('hide:popup')
        },
        handleClickCallback (event, item, index) {
            this.$emit('click:bubble', { event, item, index })
        }
    },
    watch: {
        bubblesData: {
            handler (value, oldValue) {
                value.forEach((item, index) => {
                    this.styles(item, index)
                })
            },
            immediate: true
        }
    }
}
</script>

<style>

</style>
