<script>
import moment from "moment";

export default {
    name: "TrafficCountFooter",
    props: {
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
        motDayInterval: {
            type: String,
            required: false,
            default: "5-min"
        }
    },
    data () {
        return {
            customStyle: {},
            lastUpdate: "",
            currentTabId: "infos",
            dayInterval: this.motDayInterval
        };
    },
    computed: {
        indication: function () {
            return this.$t("additional:modules.tools.gfi.themes.trafficCount.notice");
        },

        lastupdateLabel: function () {
            return this.$t("additional:modules.tools.gfi.themes.trafficCount.lastupdateLabel");
        },
    },
    watch: {
        // When the gfi window switched with arrow, the new data will be fetched from api
        thingId: {
            handler (newVal, oldVal) {
                if (oldVal) {
                    if (this.meansOfTransport !== "undefined")
                        this.setFooterLastUpdate(this.api, newVal, this.meansOfTransport);
                }
            },
            immediate: true
        },

        currentTabId: function (newVal) {
            this.setFooterLastUpdate(this.api, newVal, this.meansOfTransport);
        }
    },
    mounted: function () {
        // set the date
        if (this.meansOfTransport !== "undefined")
            this.setFooterLastUpdate(this.api, this.thingId, this.meansOfTransport, this.dayInterval);
    },
    methods: {
        /**
         * setup of the last update date
         * @param {Object} api instance of TrafficCountApi
         * @param {String} thingId the thingId to be send to any api call
         * @param {String} meansOfTransport the meansOfTransport to be send with any api call
         * @fires   Alerting#RadioTriggerAlertAlert
         * @returns {Void}  -
         */
        setFooterLastUpdate: function (api, thingId, meansOfTransport, dayInterval) {
            api.subscribeLastUpdate(thingId, meansOfTransport, dayInterval, datetime => {
                this.setLastUpdate(moment(datetime, "YYYY-MM-DD HH:mm:ss").format("DD.MM.YYYY, HH:mm:ss"));
            }, errormsg => {
                this.setLastUpdate("(aktuell keine Zeitangabe)");
                console.warn("The last update received is incomplete:", errormsg);
                Radio.trigger("Alert", "alert", {
                    content: "Das vom Sensor-Server erhaltene Datum der letzten Aktualisierung kann wegen eines API-Fehlers nicht ausgegeben werden.",
                    category: "Info"
                });
            });
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
        <div class="update">
          <p><strong>{{ lastupdateLabel }}:</strong> {{ lastUpdate }}</p>
        </div>
        <div
            class="indication"
            :style="customStyle"
        >
            {{ indication }}
        </div>
    </div>
</template>

<style lang="less" scoped>
@import "~variables";

.indication {
    font-size: 10px;
    left: 0px;
}
</style>
