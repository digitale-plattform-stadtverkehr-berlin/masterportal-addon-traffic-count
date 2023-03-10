<script>
import moment from "moment";
import TrafficCountDatePickerInput from "./TrafficCountDatePickerInput.vue";
import TrafficCountDatePickerWeek from "./TrafficCountDatePickerWeek.vue";

export default {
    name: "TrafficCountDatePicker",
    components: {
        TrafficCountDatePickerInput,
        TrafficCountDatePickerWeek
    },
    props: {
        format: {
            type: String,
            required: false,
            default: "YYYY-MM-DD"
        },
        type: {
            type: String,
            required: false,
            default: "week"
        },
        showWeekNumber: {
            type: Boolean,
            required: false,
            default: false
        },
        inputDelimiter: {
            type: String,
            required: false,
            default: ","
        },
        initialDates: {
            type: Array,
            required: false,
            default: () => []
        }
    },
    data () {
        return {
            selectedDates: [],
            inputDates: [],
            isCalendarVisible: false
        };
    },
    watch: {
        inputDates (newValue) {
            this.$emit("change", newValue);
        }
    },
    mounted () {
        this.initialDates.forEach(date => {
            this.toggleSelectedDate(moment(date, this.format));
        });
    },
    created () {
        window.addEventListener("click", (e) => {
            if (!this.$el.contains(e.target)) {
                this.isCalendarVisible = false;
            }
        });
    },
    methods: {
        /**
         * Triggered when a date picker calendar date is clicked.
         * @param {Object} momentDate A moment instance of this date.
         * @returns {void}
         */
        toggleSelectedDate (momentDate) {
            if (this.type === "week") {
                this.toggleSelectedDateWeek(momentDate);
            }
        },
        /**
         * Toggles the week of the given date.
         * @param {Object} momentDate A moment instance of this date.
         * @returns {Boolean} true if the toggle was successfull, false if not.
         */
        toggleSelectedDateWeek (momentDate) {
            if (!Array.isArray(this.selectedDates)) {
                return false;
            }
            const dayPointer = moment(momentDate);

            dayPointer.startOf("isoWeek");
            for (let i = 0; i < 7; i++) {
                this.toggleSelectedDateDay(dayPointer);
                dayPointer.add(1, "days");
            }

            this.inputDates = [];
            for (let i = 0; i < this.selectedDates.length; i++) {
                this.inputDates.push(moment(this.selectedDates[i], "YYYY-MM-DD").format(this.format));
                i += 6;
            }
            return true;
        },
        /**
         * Toggles the day of the given date.
         * @param {Object} momentDate A moment instance of this date.
         * @returns {Boolean} true if the toggle was successfull, false if not.
         */
        toggleSelectedDateDay (momentDate) {
            if (typeof momentDate?.format !== "function" || !Array.isArray(this.selectedDates)) {
                return false;
            }
            const dateStr = momentDate.format("YYYY-MM-DD"),
                idx = this.selectedDates.indexOf(dateStr);

            if (idx === -1) {
                this.selectedDates.push(dateStr);
            }
            else {
                this.selectedDates.splice(idx, 1);
            }
            return true;
        },
        /**
         * Toggles the visibility of the calendar.
         * @returns {void}
         */
        toggleCalendar () {
            this.isCalendarVisible = !this.isCalendarVisible;
        },
        /**
         * Resets selectedDates and inputDates.
         * @returns {void}
         */
        clearSelectedDates () {
            this.selectedDates = [];
            this.inputDates = [];
        }
    }
};
</script>

<template lang="html">
    <div class="datePickerContainer">
        <TrafficCountDatePickerInput
            :input-dates="inputDates"
            :delimiter="inputDelimiter"
            @toggleCalendar="toggleCalendar"
            @clearInput="clearSelectedDates"
        />
        <TrafficCountDatePickerWeek
            v-if="isCalendarVisible"
            :show-week-number="showWeekNumber"
            :selected-dates="selectedDates"
            @toggleSelectedDate="toggleSelectedDate"
        >
            <template #currentSwitch="{momentDate}">
                <slot
                    name="currentSwitch"
                    :moment-date="momentDate"
                />
            </template>
            <template #weekdayName="{weekdayName}">
                <slot
                    name="weekdayName"
                    :weekday-name="weekdayName"
                />
            </template>
            <template #weekNumber="{weekNumber}">
                <slot
                    name="weekNumber"
                    :week-number="weekNumber"
                />
            </template>
            <template #dateField="{day, momentDate, selected, inCurrentMonth}">
                <slot
                    name="dateField"
                    :day="day"
                    :moment-date="momentDate"
                    :selected="selected"
                    :in-current-month="inCurrentMonth"
                />
            </template>
        </TrafficCountDatePickerWeek>
    </div>
</template>

<style lang="scss" scoped>
    @import "~/css/mixins.scss";
    @import "~variables";

</style>
