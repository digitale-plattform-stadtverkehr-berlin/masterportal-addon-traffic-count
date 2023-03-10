<script>
import TrafficCountCompDiagram from "./TrafficCountCompDiagram.vue";
import TrafficCountCompTable from "./TrafficCountCompTable.vue";
import TrafficCountCheckbox from "./TrafficCountCheckbox.vue";
import TrafficCountDownloads from "./TrafficCountDownloads.vue";
import thousandsSeparator from "../../../../src/utils/thousandsSeparator.js";
import moment from "moment";
import DatePicker from "vue2-datepicker";
import TrafficCountDatePicker from "./TrafficCountDatePicker.vue";
import "vue2-datepicker/index.css";
import {addMissingDataWeek} from "../library/addMissingData.js";

export default {
    name: "TrafficCountWeek",
    components: {
        TrafficCountCompDiagram,
        TrafficCountCompTable,
        TrafficCountDownloads,
        TrafficCountCheckbox,
        DatePicker,
        TrafficCountDatePicker
    },
    props: {
        label: {
            type: String,
            required: true
        },
        motId: {
            type: String,
            required: true
        },
        motType: {
            type: String,
            required: true
        },
        weekStartOffset: {
            type: Number,
            required: true
        },
        api: {
            type: Object,
            required: true
        },
        thingId: {
            type: Number,
            required: true
        },
        meansOfTransport: {
            type: String,
            default: "undefined",
            required: false
        },
        archiveStartDate: {
            type: String,
            required: true
        }
    },
    data () {
        return {
            apiData: [],
            dates: null,

            // props for diagram
            setTooltipValue: (tooltipItem) => {
                return moment(tooltipItem.datetime, "YYYY-MM-DD HH:mm:ss").format("DD.MM.YYYY") + ": " + thousandsSeparator(tooltipItem.value);
            },
            yAxisTicks: 8,
            renderLabelXAxis: (datetime) => {
                return moment(datetime, "YYYY-MM-DD HH:mm:ss").format("dd");
            },
            renderLabelYAxis: (yValue) => {
                return thousandsSeparator(yValue);
            },

            // will be set on mount
            measureName: null,
            // descriptionYAxis: i18next.t("additional:modules.tools.gfi.themes.trafficCount.yAxisTextWeek"),

            renderLabelLegend: (datetime) => {
                const weeknumber = moment(datetime, "YYYY-MM-DD HH:mm:ss").week(),
                    year = moment(datetime, "YYYY-MM-DD HH:mm:ss").format("YYYY");

                return this.calendarweek + " " + weeknumber + " / " + year;
            },

            // props for table
            setTableTitle: () => {
                if (this.motType == "speed") {
                    return this.label + " " + this.measureName;
                }
                return this.label;

            },

            setColTitle: datetime => {
                return moment(datetime, "YYYY-MM-DD HH:mm:ss").format("dd");
            },
            setRowTitle: (meansOfTransports, datetime) => {
                let txt = "";

                txt += this.calendarweek + moment(datetime, "YYYY-MM-DD HH:mm:ss").format("WW");
                // for the year (YYYY) we have to add 3 days to get the thursday of the week
                txt += "/" + moment(datetime, "YYYY-MM-DD HH:mm:ss").add(3, "days").format("YYYY");
                txt += " (" + moment(datetime, "YYYY-MM-DD HH:mm:ss").format("dd, DD.MM.YYYY") + ")";
                return txt;
            },
            setFieldValue: value => {
                return thousandsSeparator(value);
            },
            weekInterval: "1-Tag",
            diagramWeekId: "diagramWeek" + this.motId,
            tableWeekId: "tableWeek" + this.motId
        };
    },
    computed: {
        calendarweek: function () {
            return this.$t("additional:modules.tools.gfi.themes.trafficCount.calendarweek");
        }
    },
    watch: {
        dates (value) {
            if (Array.isArray(value)) {
                this.weekDatepickerValueChanged(value);
            }
        }
    },
    created () {
        this.weekFormat = "YYYY [KW] WW";
        moment.locale(i18next.language);

        if (moment().isoWeekday() <= 1) {
            this.dates = [moment().subtract(7, "days").format(this.weekFormat)];
        }
        else {
            this.dates = [moment().format(this.weekFormat)];
        }
        this.setMeasureName();
    },
    methods: {
        setMeasureName: function () {
            const api = this.api,
                thingId = this.thingId,
                meansOfTransport = this.meansOfTransport,
                interval = this.weekInterval;

            api.getUnitOfMeasurement(thingId, meansOfTransport, interval, units => {
            // console.log("received units symbol "+units.symbol);
                this.measureName = units.symbol;
            }, errorUnits => {
            // console.log("received errornous symbol");
                this.measureName = "??";
            }, false, false);
        },

        /**
         * Function is initially triggered and on update
         * @param   {Date[]} dates an unsorted array of selected dates of weekday
         * @fires   Alerting#RadioTriggerAlertAlert
         * @returns {Void}  -
         */
        weekDatepickerValueChanged: function (dates) {
            const api = this.api,
                thingId = this.thingId,
                meansOfTransport = this.meansOfTransport,
                timeSettings = [];

            if (!Array.isArray(dates) || dates.length === 0) {
                this.apiData = [];
            }
            else {
                [...dates].sort((earlyDate, lateDate) => {
                    // Showing earlier date first
                    return earlyDate - lateDate;
                }).forEach(date => {
                    timeSettings.push({
                        interval: this.weekInterval,
                        from: moment(date, this.weekFormat).startOf("isoWeek").format("YYYY-MM-DD"),
                        until: moment(date, this.weekFormat).endOf("isoWeek").format("YYYY-MM-DD")
                    });
                });

                api.updateDataset(thingId, meansOfTransport, timeSettings, datasets => {
                    if (Array.isArray(datasets)) {
                        datasets.forEach((transportData, idx) => {
                            const from = typeof timeSettings[idx] === "object" ? timeSettings[idx].from + " 00:00:00" : "";

                            Object.keys(transportData).forEach(transportKey => {
                                datasets[idx][transportKey] = addMissingDataWeek(from, datasets[idx][transportKey]);
                            });
                        });
                    }

                    this.apiData = datasets;
                }, errormsg => {
                    this.apiData = [];

                    console.warn("The data received from api are incomplete:", errormsg);
                    Radio.trigger("Alert", "alert", {
                        content: "Die gewünschten Daten wurden wegen eines API-Fehlers nicht korrekt empfangen.",
                        category: "Info"
                    });
                });
            }
        },

        /**
         * Changes the dates array with given dates.
         * @param {String[]} dates The dates.
         * @returns {void}
         */
        change (dates) {
            if (!Array.isArray(dates)) {
                return;
            }
            this.dates = dates;
        },

        /**
         * Gets the date field title based on given moment date.
         * @param {Object} momentDate The moment date.
         * @returns {String} The date field title.
         */
        getDateFieldTitle (momentDate) {
            if (typeof momentDate?.format !== "function") {
                return "";
            }
            return momentDate.format("DD.MM.YYYY");
        },

        /**
         * Gets the current switch as formatted string.
         * @param {Object} momentDate A moment date.
         * @returns {String} The formatted string.
         */
        getCurrentSwitchFormatted (momentDate) {
            const months = this.$t("additional:modules.tools.gfi.themes.trafficCount.datepicker.monthsShort", {returnObjects: true});

            return `${months[momentDate.get("month")]} ${momentDate.format("YYYY")}`;
        },

        /**
         * currently unused (in branch G31DEV1-1154-trafficCount-datePickerWeek)
         * TODO: apply disabled dates when newer version appears
         * Checks if the a date should be disabled.
         * @param {Date} date The date in question.
         * @param {Date[]} currentDates The list of selected dates.
         * @returns {Boolean} true if disabled, false if enabled.
         */
        isDateDisabled (date, currentDates) {
            if (!(date instanceof Date)) {
                return true;
            }
            const endDate = moment(),
                startDate = moment("2020-01-01") > moment().subtract(1, "year") ? moment("2020-01-01") : moment().subtract(1, "year").startOf("year"),
                question = moment(date);

            if (Array.isArray(currentDates) && currentDates.length >= 5) {
                for (let i = 0; i < 5; i++) {
                    if (question.isSame(moment(currentDates[i]))) {
                        return false;
                    }
                }
                return true;
            }

            startDate.subtract(1, "days");

            return question.isSameOrBefore(startDate) || question.isSameOrAfter(endDate);
        }
    }
};
</script>

<template>
    <div>
        <h1>
            {{ label }}
        </h1>

        <div
            :id="'weekDateSelector'+motId"
            class="dateSelector"
        >
            <TrafficCountDatePicker
                type="week"
                input-delimiter=", "
                :format="weekFormat"
                :initial-dates="dates"
                :show-week-number="true"
                @change="change"
            >
                <template #currentSwitch="{momentDate}">
                    {{ getCurrentSwitchFormatted(momentDate) }}
                </template>
                <template #dateField="{day, momentDate}">
                    <span
                        :title="getDateFieldTitle(momentDate)"
                    >{{ day }}</span>
                </template>
            </TrafficCountDatePicker>
        </div>

        <TrafficCountCheckbox
            :table-diagram-id="diagramWeekId"
        />
        <div :id="diagramWeekId">
            <TrafficCountCompDiagram
                v-if="measureName"
                :api-data="apiData"
                :set-tooltip-value="setTooltipValue"
                :y-axis-ticks="yAxisTicks"
                :render-label-x-axis="renderLabelXAxis"
                :render-label-y-axis="renderLabelYAxis"
                :description-y-axis="measureName"
                :render-label-legend="renderLabelLegend"
                :do-interpolate="false"
            />
        </div>
        <TrafficCountCheckbox
            :table-diagram-id="tableWeekId"
        />
        <div :id="tableWeekId">
            <TrafficCountCompTable
                v-if="measureName"
                :api-data="apiData"
                :set-table-title="setTableTitle"
                :set-col-title="setColTitle"
                :set-row-title="setRowTitle"
                :set-field-value="setFieldValue"
            />
        </div>

        <div class="downloads">
            <TrafficCountDownloads
                :label="label"
                :set-verbose-label="setTableTitle"
                current-tab-id="week"
                :api="api"
                :thing-id="thingId"
                :means-of-transport="meansOfTransport"
            />
        </div>
    </div>
</template>

<style scoped>

h1 {margin-top:20px;}

.input-wrapper { width: 100%; }

.downloads {
  display: flex;
  justify-content: flex-start;
  margin: 0;
  flex-wrap: wrap;
}
</style>
