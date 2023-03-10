<script>
import moment from "moment";
// import ExportButtonModel from "../../../../modules/snippets/exportButton/model";
// import ExportButtonView from "../../../../modules/snippets/exportButton/view";
import ExportButtonModel from "../snippets/exportButton/model";
import ExportButtonView from "../snippets/exportButton/view";

export default {
    name: "TrafficCountDownloads",
    props: {
        label: {
          type: String,
          required: true
        },
        setVerboseLabel: {
          type: Function,
          required: true
        },
        currentTabId: {
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
        motDayInterval: {
          type: String,
          required: false,
          default: "5-min"
        },
        meansOfTransport: {
            type: String,
            default: "undefined",
            required: false
        }
    },
    data () {
        return {
            customStyle: {},
            lastUpdate: "",
            // dayInterval: "5-Min",
            dayInterval: this.motDayInterval,
            weekInterval: "1-Tag",
            yearInterval: "1-Woche",
            exportModel: new ExportButtonModel({
                tag: "Download",
                rawData: [],
                fileExtension: "csv"
            })
        };
    },
    computed: {
        exportView: function () {
            return new ExportButtonView({
                model: this.exportModel
            });
        },

        exportButtonTemplate: function () {
            return this.exportView.render().el;
        }
    },
    methods: {
        /**
         * updates the downloads of the trafficCount gfi
         * @param {String} currentTabId the id to identify the activated tab as day, week or year
         * @post the downloads is updated to show the identified tab
         * @returns {Void}  -
         */
        updateDownloads: function (currentTabId, oncomplete) {
            if (currentTabId === "day") {
                this.downloadDataDay(this.thingId, this.meansOfTransport, result => {
                    this.exportModel.set("rawData", this.prepareDataForDownload(result.data[this.meansOfTransport], this.currentTabId));
                    this.exportModel.set("tag", "Download "+this.label+" CSV");
                    this.exportModel.set("filename", result.title.replace(" ", "_") + this.setVerboseLabel());
                }, error => {
                    console.warn("error", "downloadDataDay", error);
                    Radio.trigger("Alert", "alert", {
                        content: "Die Daten können im Moment nicht heruntergeladen werden",
                        category: "Info"
                    });
                    this.exportModel.set("disabled", true);
                }, () => {
                    // onstart
                    this.exportModel.set("disabled", true);
                }, () => {
                    // oncomplete
                    this.exportModel.set("disabled", false);
                    oncomplete();
                });
            }
            else if (currentTabId === "week") {
                this.downloadDataWeek(this.thingId, this.meansOfTransport, result => {
                    this.exportModel.set("rawData", this.prepareDataForDownload(result.data[this.meansOfTransport], this.currentTabId));
                    this.exportModel.set("tag", "Download CSV "+this.label);
                    this.exportModel.set("filename", result.title.replace(" ", "_") + this.setVerboseLabel());
                    // onerror
                }, error => {
                    console.warn("error", "downloadDataWeek", error);
                    Radio.trigger("Alert", "alert", {
                        content: "Die Daten können im Moment nicht heruntergeladen werden",
                        category: "Info"
                    });
                    this.exportModel.set("disabled", true);
                }, () => {
                    // onstart
                    this.exportModel.set("disabled", true);
                }, () => {
                    // oncomplete
                    this.exportModel.set("disabled", false);
                    oncomplete();
                });
            }
            else if (currentTabId === "year") {
                this.downloadDataYear(this.thingId, this.meansOfTransport, result => {
                    this.exportModel.set("rawData", this.prepareDataForDownload(result.data[this.meansOfTransport], this.currentTabId));
                    this.exportModel.set("tag", "Download CSV "+this.label);
                    this.exportModel.set("filename", result.title.replace(" ", "_") +this.setVerboseLabel());
                    // onerror
                }, error => {
                    console.warn("error", "downloadDataYear", error);
                    Radio.trigger("Alert", "alert", {
                        content: "Die Daten können im Moment nicht heruntergeladen werden",
                        category: "Info"
                    });
                    this.exportModel.set("disabled", true);
                }, () => {
                    // onstart
                    this.exportModel.set("disabled", true);
                }, () => {
                    // oncomplete
                    this.exportModel.set("disabled", false);
                    oncomplete();
                });
            }
        },

        /**
         * gets the download data for the last 7 days for the given thingId and meansOfTransport
         * @param {Integer} thingId the ID of the thing
         * @param {String} meansOfTransport the transportation as 'Anzahl_Fahrraeder' or 'Anzahl_Kfz'
         * @param {Function} onsuccess as event function(result) with result{title, dataset} and dataset{meansOfTransport: {date: value}}; fired once on success (no subscription)
         * @param {Function} [onerror] as function(error) to fire on error
         * @param {Function} [onstart] as function() to fire before any async action has started
         * @param {Function} [oncomplete] as function() to fire after every async action no matter what
         * @returns {Void}  -
         */
        downloadDataDay: function (thingId, meansOfTransport, onsuccess, onerror, onstart, oncomplete) {
            const api = this.api,
                timeSet = {
                    interval: this.dayInterval,
                    from: moment().subtract(7, "days").format("YYYY-MM-DD"),
                    until: moment().format("YYYY-MM-DD")
                };

            api.downloadData(thingId, meansOfTransport, timeSet, onsuccess, onerror, onstart, oncomplete);
        },

        /**
         * gets the download data for the 54 weeks for the given thingId and meansOfTransport
         * @param {Integer} thingId the ID of the thing
         * @param {String} meansOfTransport the transportation as 'Anzahl_Fahrraeder' or 'Anzahl_Kfz'
         * @param {Function} onsuccess as event function(result) with result{title, dataset} and dataset{meansOfTransport: {date: value}}; fired once on success (no subscription)
         * @param {Function} [onerror] as function(error) to fire on error
         * @param {Function} [onstart] as function() to fire before any async action has started
         * @param {Function} [oncomplete] as function() to fire after every async action no matter what
         * @returns {Void}  -
         */
        downloadDataWeek: function (thingId, meansOfTransport, onsuccess, onerror, onstart, oncomplete) {
            const api = this.api,
                timeSet = {
                    interval: this.weekInterval,
                    from: moment().subtract(54, "weeks").format("YYYY-MM-DD"),
                    until: moment().format("YYYY-MM-DD")
                };

            api.downloadData(thingId, meansOfTransport, timeSet, onsuccess, onerror, onstart, oncomplete);
        },

        /**
         * gets the download data since the beginning
         * @param {Integer} thingId the ID of the thing
         * @param {String} meansOfTransport the transportation as 'Anzahl_Fahrraeder' or 'Anzahl_Kfz'
         * @param {Function} onsuccess as event function(result) with result{title, dataset} and dataset{meansOfTransport: {date: value}}; fired once on success (no subscription)
         * @param {Function} [onerror] as function(error) to fire on error
         * @param {Function} [onstart] as function() to fire before any async action has started
         * @param {Function} [oncomplete] as function() to fire after every async action no matter what
         * @returns {Void}  -
         */
        downloadDataYear: function (thingId, meansOfTransport, onsuccess, onerror, onstart, oncomplete) {
            const api = this.api;

            api.getFirstDateEver(thingId, meansOfTransport, firstDate => {
                const timeSet = {
                    interval: this.yearInterval,
                    from: firstDate,
                    until: moment().format("YYYY-MM-DD")
                };

                api.downloadData(thingId, meansOfTransport, timeSet, onsuccess, onerror, false, oncomplete);
            }, onerror, onstart, false);
        },

        /**
         * converts the data object into an array of objects for the csv download
         * @param {Object} data - the data for download
         * @param {String} tabValue - day | week | year
         * @returns {Object[]} objArr - converted data
         */
        prepareDataForDownload: function (data, tabValue) {
            const objArr = [];

            for (const key in data) {
                const obj = {},
                    date = key.split(" ");

                if (tabValue === "day") {
                    obj.Datum = date[0];
                    obj["Uhrzeit von"] = date[1].slice(0, -3);
                }
                else if (tabValue === "week") {
                    obj.Datum = date[0];
                }
                else if (tabValue === "year") {
                    obj["Kalenderwoche ab"] = date[0];
                }
                // obj.Anzahl = data[key];
                obj.Wert = data[key];
                objArr.push(obj);
            }
            return objArr;
        },

        /**
         * trigger the export function from snippet exportButton
         * @returns {Void}  -
         */
        exportFile: function () {
            this.updateDownloads(this.currentTabId, () => this.exportView.export());
        },

        /**
         * setter for lastUpdate
         * @param {String} value the datetime of the last update to be shown in the template
         * @returns {Void}  -
         */
        setLastUpdate: function (value) {
            this.lastUpdate = value;
        }
    }
};
</script>

<template>
    <div>
        <div
            v-if="currentTabId !== 'infos'"
            class="button-container"
            @click="exportFile"
            v-html="exportButtonTemplate.innerHTML"
        >
        </div>
    </div>
</template>

<style lang="scss" scoped>
@import "~variables";

.button-container {
  margin: 20px 5px;
}

</style>
