<script>

export default {
    name: "TrafficCountDatePickerInput",
    props: {
        inputDates: {
            type: Array,
            required: false,
            default: () => []
        },
        delimiter: {
            type: String,
            required: false,
            default: ","
        }
    },
    data () {
        return {
            showCalendarIcon: true
        };
    },
    computed: {
        dateComputed () {
            if (!Array.isArray(this.inputDates)) {
                return "";
            }
            return this.inputDates.join(this.delimiter);
        }
    },
    methods: {
        /**
         * Emits toggleCalendar.
         * @returns {void}
         */
        toggleCalendar () {
            this.$emit("toggleCalendar");
        },
        /**
         * Emits clearInput.
         * @returns {void}
         */
        clearInput () {
            this.$emit("clearInput");
        },
        /**
         * Sets the showCalendarIcon variable to given value.
         * @param {Boolean} value True if the calendar icon should be visible, false if not.
         * @returns {void}
         */
        setShowCalendarIcon (value) {
            this.showCalendarIcon = value;
        }
    }
};
</script>

<template>
    <div
        class="input-wrapper"
        @mouseover="setShowCalendarIcon(false)"
        @focus="setShowCalendarIcon(false)"
        @mouseleave="setShowCalendarIcon(true)"
        @blur="setShowCalendarIcon(true)"
    >
        <input
            v-model="dateComputed"
            class="date-input"
            readonly="true"
            @click="toggleCalendar"
        >
        <i :class="['bi bi-calendar4 calendar', showCalendarIcon ? 'show': '']" />
        <i
            :class="['bi bi-x', !showCalendarIcon ? 'show': '']"
            @click="clearInput"
            @keypress.enter="clearInput"
        />
    </div>
</template>

<style scoped lang="scss">
.input-wrapper {
    position: relative;
    display: inline-block;
    width: 100%;
}
.date-input {
    width: 100%;
    height: 34px;
    padding: 6px 30px;
    padding-left: 10px;
    font-size: 14px;
    line-height: 1.4;
    color: #555;
    background-color: #fff;
    border: 1px solid #ccc;
    &:hover, &:focus-within {
        outline: none;
        border-color: #409aff;
    }
}
i {
    position: absolute;
    top: 50%;
    right: 8px;
    -webkit-transform: translateY(-50%);
    transform: translateY(-50%);
    font-size: 16px;
    line-height: 1;
    color: rgba(0,0,0,.5);
    vertical-align: middle;
    display: none;
}
.bi-x {
    cursor: pointer;
}
.show {
    display: inline-block;
}
</style>
