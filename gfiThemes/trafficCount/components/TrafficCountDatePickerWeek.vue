<script>
import moment from "moment";
import TrafficCountDatePickerSwitcher from "./TrafficCountDatePickerSwitcher.vue";
import TrafficCountDatePickerCalendar from "./TrafficCountDatePickerCalendar.vue";

export default {
    name: "TrafficCountDatePickerWeek",
    components: {
        TrafficCountDatePickerSwitcher,
        TrafficCountDatePickerCalendar
    },
    props: {
        showWeekNumber: {
            type: Boolean,
            required: false,
            default: false
        },
        selectedDates: {
            type: Array,
            required: false,
            default: () => []
        }
    },
    data () {
        return {
            currentSwitch: ""
        };
    },
    created () {
        this.currentSwitch = moment().format("YYYY-MM");
    },
    methods: {
        /**
         * Triggers the toggleSelectedDate event.
         * @param {Object} date A moment object of the clicked date.
         * @returns {void}
         */
        toggleSelectedDate (date) {
            this.$emit("toggleSelectedDate", date);
        },
        /**
         * Decrements the year.
         * @returns {void}
         */
        yearDown () {
            const momentDate = moment(this.currentSwitch, "YYYY-MM");

            momentDate.subtract(1, "year");
            this.currentSwitch = momentDate.format("YYYY-MM");
        },
        /**
         * Increments the year.
         * @returns {void}
         */
        yearUp () {
            const momentDate = moment(this.currentSwitch, "YYYY-MM");

            momentDate.add(1, "year");
            this.currentSwitch = momentDate.format("YYYY-MM");
        },
        /**
         * Decrements the month.
         * @returns {void}
         */
        monthDown () {
            const momentDate = moment(this.currentSwitch, "YYYY-MM");

            momentDate.subtract(1, "month");
            this.currentSwitch = momentDate.format("YYYY-MM");
        },
        /**
         * Increments the month.
         * @returns {void}
         */
        monthUp () {
            const momentDate = moment(this.currentSwitch, "YYYY-MM");

            momentDate.add(1, "month");
            this.currentSwitch = momentDate.format("YYYY-MM");
        }
    }
};
</script>

<template>
    <div class="datePickerWeek">
        <TrafficCountDatePickerSwitcher
            :current-switch="currentSwitch"
            :show-month-selector="true"
            current-switch-format="YYYY-MM"
            @yearDown="yearDown"
            @yearUp="yearUp"
            @monthDown="monthDown"
            @monthUp="monthUp"
        >
            <template #currentSwitch="{momentDate}">
                <slot
                    name="currentSwitch"
                    :moment-date="momentDate"
                />
            </template>
        </TrafficCountDatePickerSwitcher>
        <TrafficCountDatePickerCalendar
            :current-switch="currentSwitch"
            :selected-dates="selectedDates"
            :show-week-number="showWeekNumber"
            @onClick="toggleSelectedDate"
        >
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
        </TrafficCountDatePickerCalendar>
    </div>
</template>

<style scoped lang="scss">
.datePickerWeek {
    position: absolute;
    background-color: white;
    z-index: 100;
    border: thin grey;
    box-shadow: 0 6px 12px rgb(0 0 0 / 18%);
    right: 22px;
}
</style>
