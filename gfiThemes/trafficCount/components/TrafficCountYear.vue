<script>
import TrafficCountCompDiagram from "./TrafficCountCompDiagram.vue";
import TrafficCountCompTable from "./TrafficCountCompTable.vue";
import TrafficCountCheckbox from "./TrafficCountCheckbox.vue";
import TrafficCountDownloads from "./TrafficCountDownloads.vue";
import thousandsSeparator from "../../../../src/utils/thousandsSeparator.js";
import moment from "moment";
import DatepickerModel from "../../../../modules/snippets/datepicker/model";
import DatepickerView from "../../../../modules/snippets/datepicker/view";
import {addMissingDataYear} from "../library/addMissingData.js";

export default {
    name: "TrafficCountYear",
    components: {
        TrafficCountCompDiagram,
        TrafficCountCompTable,
        TrafficCountDownloads,
        TrafficCountCheckbox
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
        },
    },
    data () {
        return {
            yearDatepicker: null,
            apiData: [],

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
            //descriptionYAxis: this.$t("additional:modules.tools.gfi.themes.trafficCount.yAxisTextYear"),

            renderLabelLegend: (datetime) => {
                return moment(datetime, "YYYY-MM-DD HH:mm:ss").add(3, "days").format("YYYY");
            },

            // props for table
            setTableTitle: () => {
                if (this.motType == "speed") {
                    return this.label + " " + this.measureName;
                } else {
                    return this.label;
                }
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
            diagramYearId: "diagramYear"+this.motId,
            tableYearId: "tableYear"+this.motId
        };
    },
    mounted () {
        if (this.meansOfTransport !== "undefined")
            this.setYearDatepicker();
        this.setMeasureName();
    },
    methods: {
      setMeasureName: function() {
        const api = this.api;
        const thingId = this.thingId;
        const meansOfTransport = this.meansOfTransport;
        const interval = this.yearInterval;

        api.getUnitOfMeasurement(thingId, meansOfTransport, interval, units => {
          // console.log("received units symbol "+units.symbol);
          this.measureName = units.symbol;
        }, errorUnits => {
          // console.log("received errornous symbol");
          this.measureName = "??";
        }, false, false);
      },

        /**
         * Setup of the year tab.
         * This methode creates a datepicker model and triggers the view for rendering. Snippets must be added after view.render.
         * @listens Snippets#ValuesChanged
         * @returns {Void}  -
         */
        setYearDatepicker: function () {
            const startDate = moment(this.archiveStartDate);

            // create datepicker only on first enter of tab
            if (!this.yearDatepicker) {
                this.yearDatepicker = new DatepickerModel({
                    displayName: "Tag",
                    preselectedValue: moment().startOf("year").toDate(),
                    multidate: 5,
                    startDate: startDate.toDate(),
                    endDate: moment().startOf("year").toDate(),
                    type: "datepicker",
                    minViewMode: "years",
                    maxViewMode: "years",
                    inputs: $(document.getElementById("yearDateInput"+this.motId)),
                    format: "yyyy",
                    language: i18next.language
                });

                this.yearDatepicker.on("valuesChanged", function (evt) {
                    let date = evt.attributes.date;

                    if (date && !Array.isArray(date)) {
                        date = [date];
                    }
                    this.yearDatepickerValueChanged(date);
                }.bind(this));

                if (document.querySelector("#yearDateSelector"+this.motId)) {
                    document.querySelector("#yearDateSelector"+this.motId).appendChild(new DatepickerView({model: this.yearDatepicker}).render().el);
                }
                this.yearDatepicker.updateValues(moment().toDate());
            }
            else if (document.querySelector("#yearDateSelector"+this.motId)) {
                document.querySelector("#yearDateSelector"+this.motId).appendChild(new DatepickerView({model: this.yearDatepicker}).render().el);
            }
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

            if (dates.length === 0) {
                this.apiData = [];
            }
            else {
                dates.sort((earlyDate, lateDate) => {
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
         * opens the calender
         * @returns {void}
         */
        toggleCalendar: function () {
            const input = this.$el.querySelector("input");

            input.focus();
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
            <div class="input-group">
                <input
                    :id="'yearDateInput'+motId"
                    type="text"
                    class="form-control dpinput"
                    placeholder="Datum"
                >
                <span class="input-group-btn">
                    <button
                        :id="'yearDateInputButton'+motId"
                        class="btn btn-default yearDateInputButton"
                        type="button"
                        @click="toggleCalendar"
                    >
                        <span
                            class="glyphicon glyphicon-th"
                            aria-hidden="true"
                        ></span>
                    </button>
                </span>
            </div>
        </div>

        <TrafficCountCheckbox
            :tableDiagramId="diagramYearId"
        />
        <div :id="diagramYearId">
            <TrafficCountCompDiagram
              v-if='measureName'
                :apiData="apiData"
                :setTooltipValue="setTooltipValue"
                :yAxisTicks="yAxisTicks"
                :renderLabelXAxis="renderLabelXAxis"
                :renderLabelYAxis="renderLabelYAxis"
                :descriptionYAxis="measureName"
                :renderLabelLegend="renderLabelLegend"
                :doInterpolate="false"
            />
        </div>
        <TrafficCountCheckbox
            :tableDiagramId="tableYearId"
        />
        <div :id="tableYearId">
            <TrafficCountCompTable
              v-if='measureName'
                :apiData="apiData"
                :setTableTitle="setTableTitle"
                :setColTitle="setColTitle"
                :setRowTitle="setRowTitle"
                :setFieldValue="setFieldValue"
            />
        </div>

        <div class="downloads">
          <TrafficCountDownloads
              :label="label"
              :setVerboseLabel="setTableTitle"
              currentTabId="year"
              :api="api"
              :thingId="thingId"
              :meansOfTransport="meansOfTransport"
          />
        </div>

    </div>
</template>

<style scoped>
.yearDateInputButton{
    padding: 6px 12px 5px 12px
}
.downloads {
  display: flex;
  justify-content: flex-start;
  margin: 0;
  flex-wrap: wrap;
}
</style>
