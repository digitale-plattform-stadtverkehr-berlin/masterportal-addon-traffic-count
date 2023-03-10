<script>
import TrafficCountCompDiagram from "./TrafficCountCompDiagram.vue";
import TrafficCountCompTable from "./TrafficCountCompTable.vue";
import TrafficCountCheckbox from "./TrafficCountCheckbox.vue";
import TrafficCountDownloads from "./TrafficCountDownloads.vue";
import thousandsSeparator from "../../../../src/utils/thousandsSeparator.js";
import moment from "moment";
import DatePicker from "vue2-datepicker";
import "vue2-datepicker/index.css";
import {addMissingDataYear} from "../library/addMissingData.js";

export default {
    name: "TrafficCountYear",
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
            dates: [],

            // props for diagram
            setTooltipValue: (tooltipItem) => {
                // add 3 days to match thursdays
                const objMoment = moment(tooltipItem.datetime, "YYYY-MM-DD HH:mm:ss").add(3, "days");

                return this.$t("additional:modules.tools.gfi.themes.trafficCount.calendarweek") + " " + objMoment.format("WW") + " / " + objMoment.format("YYYY") + ": " + thousandsSeparator(tooltipItem.value);
            },
            yAxisTicks: 8,
            renderLabelXAxis: (datetime) => {
                // add 3 days to match thursdays
                const objMoment = moment(datetime, "YYYY-MM-DD HH:mm:ss").add(3, "days");

                return this.$t("additional:modules.tools.gfi.themes.trafficCount.calendarweek") + objMoment.format("WW");
            },
            renderLabelYAxis: (yValue) => {
                return thousandsSeparator(yValue);
            },

            // will be set on mount
            measureName: null,
            // descriptionYAxis: this.$t("additional:modules.tools.gfi.themes.trafficCount.yAxisTextYear"),

            renderLabelLegend: (datetime) => {
                return moment(datetime, "YYYY-MM-DD HH:mm:ss").add(3, "days").format("YYYY");
            },

            // props for table
            setTableTitle: () => {
                if (this.motType == "speed") {
                    return this.label + " " + this.measureName;
                }
                return this.label;

            },

            setColTitle: datetime => {
                return this.$t("additional:modules.tools.gfi.themes.trafficCount.calendarweek") + moment(datetime, "YYYY-MM-DD HH:mm:ss").format("WW");
            },
            setRowTitle: (meansOfTransports, datetime) => {
                // datetime is the monday of the week - so we have to add 3 days to get the thursday of the week
                const txt = moment(datetime, "YYYY-MM-DD HH:mm:ss").add(3, "days").format("YYYY");

                return txt;
            },
            setFieldValue: value => {
                return thousandsSeparator(value);
            },
            yearInterval: "1-Woche",
            diagramYearId: "diagramYear" + this.motId,
            tableYearId: "tableYear" + this.motId
        };
    },
    watch: {
        dates (value) {
            this.yearDatepickerValueChanged(value);
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
                interval = this.yearInterval;

            api.getUnitOfMeasurement(thingId, meansOfTransport, interval, units => {
                // console.log("received units symbol "+units.symbol);
                this.measureName = units.symbol;
            }, errorUnits => {
                // console.log("received errornous symbol");
                this.measureName = "??";
            }, false, false);
        },

        /** Function is initially triggered and on update
         * @param   {Date} dates an unsorted array of first day date of selected year
         * @fires   Alerting#RadioTriggerAlertAlert
         * @returns {void}
         */
        yearDatepickerValueChanged: function (dates) {
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
                        interval: this.yearInterval,
                        // subtract 3 days to savely include the first thursday of january into the interval, as the first calendar week always includes the first thursday of january
                        from: moment(date).startOf("year").subtract(3, "days").format("YYYY-MM-DD"),
                        // add 3 days to savely include the last thursday of december into the interval, as the last calendar week always includes the last thursday of december
                        until: moment(date).endOf("year").add(3, "days").format("YYYY-MM-DD"),
                        selectedYear: moment(date).format("YYYY")
                    });
                });

                api.updateDataset(thingId, meansOfTransport, timeSettings, datasets => {
                    if (Array.isArray(datasets)) {
                        datasets.forEach((transportData, idx) => {
                            const from = typeof timeSettings[idx] === "object" ? timeSettings[idx].selectedYear : "";

                            Object.keys(transportData).forEach(transportKey => {
                                datasets[idx][transportKey] = addMissingDataYear(from, datasets[idx][transportKey]);
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
         * Checks if the a date should be disabled.
         * @param {Date} date The date in question.
         * @param {Date[]} currentDates The list of selected dates.
         * @returns {Boolean} true if disabled, false if enabled.
         */
        isDateDisabled (date, currentDates) {
            if (!(date instanceof Date)) {
                return true;
            }
            const endDate = this.checkGurlittInsel ? moment().subtract(1, "days") : moment(),
                startMoment = moment().startOf("year").subtract(10, "years"),
                startYear = parseInt(startMoment.format("YYYY"), 10),
                question = moment(date);

            if (startYear < 2015) {
                startMoment.add(2015 - startYear, "years");
            }

            startMoment.subtract(1, "year");

            if (Array.isArray(currentDates) && currentDates.length >= 5) {
                for (let i = 0; i < 5; i++) {
                    if (question.isSame(moment(currentDates[i]))) {
                        return false;
                    }
                }
                return true;
            }

            return question.isSameOrBefore(startMoment) || question.isSameOrAfter(endDate);
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
            :id="'yearDateSelector'+motId"
            class="dateSelector"
        >
            <DatePicker
                v-model="dates"
                aria-label="Datum"
                placeholder="Datum"
                type="year"
                format="YYYY"
                :multiple="true"
                :disabled-date="isDateDisabled"
                title-format="YYYY"
                :lang="$t('common:libraries.vue2-datepicker.lang', {returnObjects: true})"
            />
        </div>

        <TrafficCountCheckbox
            :table-diagram-id="diagramYearId"
        />
        <div :id="diagramYearId">
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
            :table-diagram-id="tableYearId"
        />
        <div :id="tableYearId">
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
                current-tab-id="year"
                :api="api"
                :thing-id="thingId"
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
