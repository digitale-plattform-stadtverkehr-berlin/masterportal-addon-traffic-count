<script>
import moment from "moment";

export default {
    name: "TrafficCountDatePickerCalendar",
    props: {
        currentSwitch: {
            type: String,
            required: false,
            default: ""
        },
        selectedDates: {
            type: Array,
            required: false,
            default () {
                return [];
            }
        },
        showWeekNumber: {
            type: Boolean,
            required: false,
            default: false
        }
    },
    data () {
        return {
            currentDates: []
        };
    },
    computed: {
        namesOfWeekdaysComputed () {
            const result = [],
                weekdays = this.$t("additional:modules.tools.gfi.themes.trafficCount.datepicker.weekdaysShort", {returnObjects: true});

            for (let i = 1; i <= 7; i++) {
                result.push(weekdays[i % 7]);
            }
            return result;
        }
    },
    watch: {
        currentSwitch (val) {
            this.currentDates = this.getCurrentDates(val, this.selectedDates);
        },
        selectedDates (val) {
            this.currentDates = this.getCurrentDates(this.currentSwitch, val);
        }
    },
    mounted () {
        this.currentDates = this.getCurrentDates(this.currentSwitch, this.selectedDates);
    },
    methods: {
        /**
         * Returns the current dates to build the template with.
         * @param {String} currentSwitch The current switch command to identify the month and the year.
         * @param {String[]} selectedDates An array of previous selected dates.
         * @returns {Object[]} An array of Objects to build the template with.
         */
        getCurrentDates (currentSwitch, selectedDates) {
            const result = [],
                currentMoment = moment(currentSwitch, "YYYY-MM"),
                datePointer = currentMoment.isValid() ? moment(currentMoment) : moment(),
                endDate = moment(datePointer);

            datePointer.startOf("isoWeek");
            endDate.startOf("isoWeek").add(41, "days");

            while (datePointer.isSameOrBefore(endDate)) {
                if (datePointer.isoWeekday() === 1) {
                    result.push({
                        weekNumber: datePointer.format("W"),
                        days: []
                    });
                }
                result[result.length - 1].days.push({
                    day: datePointer.format("D"),
                    momentDate: moment(datePointer),
                    selected: selectedDates.includes(datePointer.format("YYYY-MM-DD")),
                    inCurrentMonth: currentMoment.format("M") === datePointer.format("M")
                });
                datePointer.add(1, "days");
            }
            return result;
        },
        /**
         * Emits on click.
         * @param {Object} momentDate A moment object of the clicked date.
         * @returns {void}
         */
        onClick (momentDate) {
            this.$emit("onClick", momentDate);
        }
    }
};
</script>

<template lang="html">
    <div class="datePickerCalendarContainer">
        <table>
            <tr>
                <th v-if="showWeekNumber" />
                <th
                    v-for="(weekdayName, index) in namesOfWeekdaysComputed"
                    :key="'weekdayName-' + index"
                    class="weekdayName"
                >
                    <slot
                        name="weekdayName"
                        :weekday-name="weekdayName"
                    >
                        {{ weekdayName }}
                    </slot>
                </th>
            </tr>
            <tr
                v-for="dateRow in currentDates"
                :key="'dateRow-' + dateRow.weekNumber"
            >
                <td
                    v-if="showWeekNumber"
                    class="weekNumber"
                >
                    <slot
                        name="weekNumber"
                        :week-number="dateRow.weekNumber"
                    >
                        {{ dateRow.weekNumber }}
                    </slot>
                </td>
                <td
                    v-for="dateObj in dateRow.days"
                    :key="'dateObj-' + dateObj.day"
                    :class="{dateField: true, outOfMonth: !dateObj.inCurrentMonth, selected: dateObj.selected}"
                    @click="onClick(dateObj.momentDate)"
                    @keypress.enter="onClick(dateObj.momentDate)"
                >
                    <slot
                        name="dateField"
                        :day="dateObj.day"
                        :moment-date="dateObj.momentDate"
                        :selected="dateObj.selected"
                        :in-current-month="dateObj.inCurrentMonth"
                    >
                        {{ dateObj.day }}
                    </slot>
                </td>
            </tr>
        </table>
    </div>
</template>

<style lang="scss" scoped>
    @import "~/css/mixins.scss";
    @import "~variables";

    .datePickerCalendarContainer {
        td, th {
            width: 32px;
            height: 32px;
            text-align: center;
            vertical-align: middle;
        }
        th {
            font-weight: normal;
        }
        td {
            &.dateField {
                cursor: pointer;
            }
            &.outOfMonth {
                color: $light_grey;
            }
            &.selected {
                color: $white;
                background-color: $light_blue;
            }
        }
    }
</style>
