<script>
import TrafficCountCompDiagram from "./TrafficCountCompDiagram.vue";
import TrafficCountCompTable from "./TrafficCountCompTable.vue";
import TrafficCountCheckbox from "./TrafficCountCheckbox.vue";
import TrafficCountDownloads from "./TrafficCountDownloads.vue";
import thousandsSeparator from "../../../../src/utils/thousandsSeparator.js";
import moment from "moment";

import DatePicker from "vue2-datepicker";
import "vue2-datepicker/index.css";

import {addMissingDataDay} from "../library/addMissingData.js";

export default {
    name: "TrafficCountDay",
    components: {
        TrafficCountCompDiagram,
        TrafficCountCompTable,
        TrafficCountDownloads,
        TrafficCountCheckbox,
        DatePicker
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
        motDayInterval: {
            type: String,
            required: false,
            default: "5-min"
        },
        dayStartOffset: {
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
            // default: "undefined",
            required: true
        }
    },
    data () {
        return {
            dayDatepicker: null,
            apiData: [],
            dates: [],

            // props for diagram
            setTooltipValue: (tooltipItem) => {
                return moment(tooltipItem.datetime, "YYYY-MM-DD HH:mm:ss").format("HH:mm") + " " + this.$t("additional:modules.tools.gfi.themes.trafficCount.clockLabel") + ": " + thousandsSeparator(tooltipItem.value);
            },
            xAxisTicks: 12,
            yAxisTicks: 8,
            renderLabelXAxis: (datetime) => {
                return moment(datetime, "YYYY-MM-DD HH:mm:ss").format("HH:mm") + " " + this.$t("additional:modules.tools.gfi.themes.trafficCount.clockLabel");
            },
            renderLabelYAxis: (yValue) => {
                return thousandsSeparator(yValue);
            },

            // will be set on mount
            measureName: null, // this.$t("additional:modules.tools.gfi.themes.trafficCount.yAxisTextDay");

            renderLabelLegend: (datetime) => {
                return moment(datetime, "YYYY-MM-DD HH:mm:ss").format("DD.MM.YYYY");
            },

            // props for table
            setTableTitle: () => {
                if (this.motType == "speed") {
                    return this.label + " " + this.measureName;
                }
                return this.label;

            },
            setColTitle: datetime => {
                return moment(datetime, "YYYY-MM-DD HH:mm:ss").format("HH:mm") + " " + this.$t("additional:modules.tools.gfi.themes.trafficCount.clockLabel");
            },
            setRowTitle: (meansOfTransports, datetime) => {
                const txt = moment(datetime, "YYYY-MM-DD HH:mm:ss").format("DD.MM.YYYY");

                return txt;
            },
            setFieldValue: value => {
                return thousandsSeparator(value);
            },
            // dayInterval: "5-Min",
            dayInterval: this.motDayInterval,
            diagramDayId: "diagramDay" + this.motId,
            tableDayId: "tableDay" + this.motId
        };
    },
    watch: {
        dates (value) {
            this.dayDatepickerValueChanged(value);
        }
    },
    mounted () {
        moment.locale(i18next.language);
        this.initializeDates();
        this.setMeasureName();
    },
    methods: {
        initializeDates () {
            this.dates = [moment().toDate()];
        },

        setMeasureName: function () {
            const api = this.api,
                thingId = this.thingId,
                meansOfTransport = this.meansOfTransport,
                interval = this.dayInterval;

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
        dayDatepickerValueChanged: function (dates) {
            const api = this.api,
                thingId = this.thingId,
                meansOfTransport = this.meansOfTransport,
                timeSettings = [];

            if (dates) {
                if (!Array.isArray(dates) || dates.length === 0) {
                    this.apiData = [];
                }
                else {
                    [...dates].sort((earlyDate, lateDate) => {
                        // Showing earlier date first
                        return earlyDate - lateDate;
                    }).forEach(date => {
                        const fromDate = moment(date).format("YYYY-MM-DD");

                        timeSettings.push({
                            interval: this.dayInterval,
                            from: fromDate,
                            until: fromDate
                        });
                    });

                    api.updateDataset(thingId, meansOfTransport, timeSettings, datasets => {
                        if (Array.isArray(datasets)) {
                            datasets.forEach((transportData, idx) => {
                                const from = typeof timeSettings[idx] === "object" ? timeSettings[idx].from + " 00:00:00" : "";

                                Object.keys(transportData).forEach(transportKey => {
                                    datasets[idx][transportKey] = addMissingDataDay(from, datasets[idx][transportKey], parseInt(this.dayInterval));
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
            }
        },

        /**
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
                startDate = moment().subtract(7, "days"),
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
        <h1>{{ label }}</h1>
        <div
            :id="'dayDateSelector'+motId"
            class="dateSelector"
        >
            <DatePicker
                v-model="dates"
                aria-label="Datum"
                placeholder="Datum"
                type="date"
                format="DD.MM.YYYY"
                :multiple="true"
                :show-week-number="true"
                :disabled-date="isDateDisabled"
                title-format="DD.MM.YYYY"
                :lang="$t('common:libraries.vue2-datepicker.lang', {returnObjects: true})"
            />
        </div>
        <TrafficCountCheckbox :table-diagram-id="diagramDayId" />
        <div :id="diagramDayId">
            <TrafficCountCompDiagram
                v-if="measureName"
                :api-data="apiData"
                :set-tooltip-value="setTooltipValue"
                :x-axis-ticks="xAxisTicks"
                :y-axis-ticks="yAxisTicks"
                :render-label-x-axis="renderLabelXAxis"
                :render-label-y-axis="renderLabelYAxis"
                :description-y-axis="measureName"
                :render-label-legend="renderLabelLegend"
            />
        </div>
        <TrafficCountCheckbox
            :table-diagram-id="tableDayId"
        />
        <div :id="tableDayId">
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
                current-tab-id="day"
                :api="api"
                :thing-id="thingId"
                :mot-day-interval="motDayInterval"
                :means-of-transport="meansOfTransport"
            />
        </div>
    </div>
</template>

<style scoped>

h1 {margin-top:20px;}

.mx-datepicker { width: 100%; }

.downloads {
  display: flex;
  justify-content: flex-start;
  margin: 0;
  flex-wrap: wrap;
}
</style>
