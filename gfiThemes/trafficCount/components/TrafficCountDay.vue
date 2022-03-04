<script>
import TrafficCountCompDiagram from "./TrafficCountCompDiagram.vue";
import TrafficCountCompTable from "./TrafficCountCompTable.vue";
import TrafficCountCheckbox from "./TrafficCountCheckbox.vue";
import TrafficCountDownloads from "./TrafficCountDownloads.vue";
import thousandsSeparator from "../../../../src/utils/thousandsSeparator.js";
import moment from "moment";
import DatepickerModel from "../../../../modules/snippets/datepicker/model";
import DatepickerView from "../../../../modules/snippets/datepicker/view";
import {addMissingDataDay} from "../library/addMissingData.js";

export default {
    name: "TrafficCountDay",
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
        motDayInterval: {
          type: String,
          required: false,
          default: "5-min"
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
        }
    },
    data () {
        return {
            dayDatepicker: null,
            apiData: [],

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
            measureName: null,  //this.$t("additional:modules.tools.gfi.themes.trafficCount.yAxisTextDay");

            renderLabelLegend: (datetime) => {
                return moment(datetime, "YYYY-MM-DD HH:mm:ss").format("DD.MM.YYYY");
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
            diagramDayId: "diagramDay"+this.motId,
            tableDayId: "tableDay"+this.motId
        };
    },
    mounted () {
        moment.locale(i18next.language);

        if (this.meansOfTransport !== "undefined")
            this.setDayDatepicker();
        this.setMeasureName();
    },
    methods: {
        setMeasureName: function() {
          const api = this.api;
          const thingId = this.thingId;
          const meansOfTransport = this.meansOfTransport;
          const interval = this.dayInterval;

          api.getUnitOfMeasurement(thingId, meansOfTransport, interval, units => {
            // console.log("received units symbol "+units.symbol);
            this.measureName = units.symbol;
          }, errorUnits => {
            // console.log("received errornous symbol");
            this.measureName = "??";
          }, false, false);
        },

        setDayDatepicker: function () {
            const startDate = moment().subtract(7, "days");

            if (!this.dayDatepicker) {
                this.dayDatepicker = new DatepickerModel({
                    displayName: "Tag",
                    multidate: 5,
                    preselectedValue: moment().toDate(),
                    startDate: startDate.toDate(),
                    endDate: moment().toDate(),
                    type: "datepicker",
                    inputs: $(document.getElementById("dayDateInput"+this.motId)),
                    todayHighlight: false,
                    language: i18next.language
                });

                this.dayDatepicker.on("valuesChanged", function (evt) {
                    let date = evt.attributes.date;

                    if (date && !Array.isArray(date)) {
                        date = [date];
                    }
                    this.dayDatepickerValueChanged(date);
                }.bind(this));

                if (document.querySelector("#dayDateSelector"+this.motId)) {
                    document.querySelector("#dayDateSelector"+this.motId).appendChild(new DatepickerView({model: this.dayDatepicker}).render().el);
                }
                this.dayDatepicker.updateValues(moment().toDate());
            }
            else if (document.querySelector("#dayDateSelector"+this.motId)) {
                document.querySelector("#dayDateSelector"+this.motId).appendChild(new DatepickerView({model: this.dayDatepicker}).render().el);
            }
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

            if (dates.length === 0) {
                this.apiData = [];
            }
            else {
                dates.sort((earlyDate, lateDate) => {
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
        },

        /**
         * opens the calender
         * @returns {Void}  -
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
            :id="'dayDateSelector'+motId"
            class="dateSelector"
        >
            <div class="input-group">
                <input
                    :id="'dayDateInput'+motId"
                    type="text"
                    class="form-control dpinput"
                    placeholder="Datum"
                >
                <span class="input-group-btn">
                    <button
                        :id="'dayDateInputButton'+motId"
                        class="btn btn-default dayDateInputButton"
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
            :tableDiagramId="diagramDayId"
        />
        <div :id="diagramDayId">
            <TrafficCountCompDiagram
              v-if='measureName'
                :apiData="apiData"
                :setTooltipValue="setTooltipValue"
                :xAxisTicks="xAxisTicks"
                :yAxisTicks="yAxisTicks"
                :renderLabelXAxis="renderLabelXAxis"
                :renderLabelYAxis="renderLabelYAxis"
                :descriptionYAxis="measureName"
                :renderLabelLegend="renderLabelLegend"
            />
        </div>
        <TrafficCountCheckbox
            :tableDiagramId="tableDayId"
        />
        <div :id="tableDayId">
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
              currentTabId="day"
              :api="api"
              :thingId="thingId"
              :motDayInterval="motDayInterval"
              :meansOfTransport="meansOfTransport"
          />
        </div>

    </div>
</template>

<style scoped>
.dayDateInputButton{
  padding: 6px 12px 5px 12px
}
.downloads {
  display: flex;
  justify-content: flex-start;
  margin: 0;
  flex-wrap: wrap;
}
</style>
