<script>
import {mapGetters} from "vuex";

import {TrafficCountCache} from "../library/trafficCountCache";
import {TrafficCountApi} from "../library/trafficCountApi";
import TrafficCountInfo from "./TrafficCountInfo.vue";
import TrafficCountDay from "./TrafficCountDay.vue";
import TrafficCountWeek from "./TrafficCountWeek.vue";
import TrafficCountYear from "./TrafficCountYear.vue";
import TrafficCountFooter from "./TrafficCountFooter.vue";

export default {
    name: "TrafficCount",
    components: {
        TrafficCountInfo,
        TrafficCountDay,
        TrafficCountWeek,
        TrafficCountYear,
        TrafficCountFooter
    },
    props: {
        feature: {
            type: Object,
            required: true
        }
    },
    data () {
        return {
            api: null,
            propThingId: 0,
            meansOfTransports: [],
            meansOfTransportsCount: [],
            meansOfTransportsSpeed: [],
            title: "",
            description: "",
            headerProperties: [],
            hasPhotos: false,
            headerPhotosUrls: [],
            archiveStartDate: "2020-01-01",
            dayStartOffset: 0,
            weekStartOffset: 1,
            meansOfTransport: "",
            currentTabId: "infos",
            keyInfo: "info",
            keyDay: "day",
            keyWeek: "week",
            keyYear: "year"
        };
    },
    computed: {
        ...mapGetters("Language", ["currentLocale"]),
        infoLabel: function () {
            return this.$t("additional:modules.tools.gfi.themes.trafficCount.infoLabel");
        },

        dayLabel: function () {
            return this.$t("additional:modules.tools.gfi.themes.trafficCount.dayLabel");
        },

        weekLabel: function () {
            return this.$t("additional:modules.tools.gfi.themes.trafficCount.weekLabel");
        },

        yearLabel: function () {
            return this.$t("additional:modules.tools.gfi.themes.trafficCount.yearLabel");
        },

        idLabel: function () {
            return this.$t("additional:modules.tools.gfi.themes.trafficCount.idLabel");
        },

        meansOfTransportLabel: function () {
            return this.$t("additional:modules.tools.gfi.themes.trafficCount.meansOfTransportLabel");
        },

        descriptionLabel: function () {
            return this.$t("additional:modules.tools.gfi.themes.trafficCount.descriptionLabel");
        }
    },
    watch: {
        // When the gfi window switched with arrow, the connection will be refreshed
        feature: {
            handler (newVal, oldVal) {
                if (oldVal) {
                    this.createDataConnection(newVal.getProperties(), null).then(() => {
                        this.setHeader(this.api, this.propThingId, this.meansOfTransportsCount[0]);
                        this.setComponentKey(this.propThingId);
                        this.setActiveDefaultTab();
                    });
                }
            },
            immediate: true
        },

        // When language is switched, the header will be rerendered
        currentLocale: function (newVal, oldVal) {
            if (oldVal) {
                // this.setHeader(this.api, this.propThingId, this.propMeansOfTransportKfz);
                this.setHeader(this.api, this.propThingId, this.meansOfTransportsCount[0]);
                this.setComponentKey(newVal);
                this.setActiveDefaultTab();
            }
        }
    },
    created: function () {
        const gfiTheme = this.feature?.getTheme(),
            gfiParams = gfiTheme?.params;

        this.meansOfTransports = typeof gfiParams === "object" && gfiParams.hasOwnProperty("meansOfTransports") ? gfiParams.meansOfTransports : [];
        this.createDataConnection(this.feature.getProperties(), null).then(() => {
            this.meansOfTransportsCount = this.meansOfTransports.filter(function isCount (mot) {
                return mot.type == "counting";
            });
            this.meansOfTransportsSpeed = this.meansOfTransports.filter(function isSpeed (mot) {
                return mot.type == "speed";
            });

            const headerPropertyNames = typeof gfiParams === "object" && gfiParams.hasOwnProperty("headerProperties") ? gfiParams.headerProperties : [];

            headerPropertyNames.forEach(hpName => {
                const headerProperty = {
                    name: hpName,
                    label: this.$t("common:trafficCountProperties." + hpName),
                    value: ""
                };

                this.headerProperties.push(headerProperty);
            });

            // get special property "photos" from properties
            if (typeof gfiParams === "object" && gfiParams.hasOwnProperty("headerPhotos") && gfiParams.headerPhotos) {
                this.hasPhotos = true;
            }

            this.archiveStartDate = typeof gfiParams === "object" && gfiParams.hasOwnProperty("archiveStartDate") ? gfiParams.archiveStartDate : this.archiveStartDate;

            this.dayStartOffset = typeof gfiParams === "object" && gfiParams.hasOwnProperty("dayStartOffset") ? gfiParams.dayStartOffset : this.dayStartOffset;

            this.weekStartOffset = typeof gfiParams === "object" && gfiParams.hasOwnProperty("weekStartOffset") ? gfiParams.weekStartOffset : this.weekStartOffset;

            this.setHeader(this.api, this.propThingId, this.meansOfTransportsCount[0], this.headerProperties);
            this.setComponentKey(this.propThingId);
            this.setActiveDefaultTab();
        });
    },
    mounted: function () {
        this.setHeader(this.api, this.propThingId, this.meansOfTransportsCount[0], this.headerProperties);
        this.setGfiDiagramWidth(); // init size
    },
    beforeDestroy: function () {
        this.api.unsubscribeEverything();
    },
    methods: {
        /**
         * it will make conntection to thing api
         * @param {Object[]} feature the feature properties from thing
         * @param {Object} [sensorThingsApiOpt=null] an optional api for testing
         * @returns {Void} -
         */
        createDataConnection: function (feature, sensorThingsApiOpt = null) {
            return new Promise((resolve) => {
                const thingId = feature["@iot.id"],
                    url = feature.requestUrl,
                    sensorThingsApiVersion = "v" + feature.versionUrl,
                    mqttOptions = {
                        mqttUrl: "wss://" + url.split("/")[2] + "/mqtt",
                        mqttVersion: "3.1.1",
                        rhPath: url,
                        context: this
                    };

                this.api = new TrafficCountCache(url, sensorThingsApiVersion, mqttOptions, sensorThingsApiOpt);
                this.propThingId = thingId;

                const tcApi = new TrafficCountApi(url, sensorThingsApiVersion, mqttOptions, sensorThingsApiOpt),
                    interval = this.meansOfTransports[0].dayInterval;

                tcApi.getBaseDataStreams(thingId, interval, datastreams => {
                    feature.Datastreams = datastreams;
                }, false, false, () => {
                    for (let i = 0; i < this.meansOfTransports.length; i++) {
                        this.meansOfTransports[i].stream =
                            this.getMeansOfTransportFromDatastream(feature.Datastreams, [this.meansOfTransports[i].id]);
                    }
                    resolve();

                });

            });
        },

        /**
         * returns the value in meansOfTransportArray that matches the start of the given array of datastreams property layerName, returns first match
         * @param {Object[]} datastreams the array of datastreams from the SensorThingsAPI
         * @param {String[]} meansOfTransportArray an array representing all terms to look for in the datastreams layerName
         * @returns {String|Boolean}  a string representing the means of transport (e.g. Anzahl_Kfz, Anzahl_Fahrraeder) or false if no means of transport where found
         */
        getMeansOfTransportFromDatastream: function (datastreams, meansOfTransportArray) {
            let key,
                i,
                datastream = null;

            if (!Array.isArray(datastreams) || datastreams.length === 0) {
                return false;
            }

            for (i in datastreams) {
                datastream = datastreams[i];

                if (!datastream || typeof datastream !== "object" || !datastream.hasOwnProperty("properties") || !datastream.properties.hasOwnProperty("layerName")) {
                    continue;
                }

                for (key in meansOfTransportArray) {
                    if (datastream.properties.layerName.indexOf(meansOfTransportArray[key]) === 0) {
                        return meansOfTransportArray[key];
                    }
                }
            }

            return false;
        },

        /**
         * set the default infos tab active when switch the language by triggering the click event
         * @returns {Void} -
         */
        setActiveDefaultTab: function () {
            this.$el.querySelector("li[value='infos'] a").click();
        },

        /**
         * set the current tab id after clicking.
         * @param {Object[]} evt the target of current click event
         * @returns {Void} -
         */
        setCurrentTabId: function (evt) {
            if (evt && evt.target && evt.target.hash) {
                this.currentTabId = evt.target.hash.substring(1);
            }
        },

        /**
         * set the header of gfi theme
         * @param {Object} api the api from library
         * @param {String} thingId the current thing Id
         * @param {String} meansOfTransport the means of transportation
         * @param {Array} headerProperties the properties for the header
         * @returns {Void} -
         */
        setHeader: function (api, thingId, meansOfTransport, headerProperties) {
            // title
            api.updateTitle(thingId, title => {
                this.setTitle(title);
            }, errormsg => {
                this.setTitle("(nicht bekannt)");
                console.warn("The title received is incomplete:", errormsg);
            });

            // description
            api.updateDescription(thingId, description => {
                this.setDescription(description);
            }, errormsg => {
                this.setDescription("(nicht bekannt)");
                console.warn("The description received is incomplete:", errormsg);
            });

            // values of header properties
            this.headerProperties.forEach(headerProperty => {
                api.updateProperty(thingId, headerProperty.name, hpValue => {
                    this.setHeaderProperty(headerProperty.name, hpValue);
                }, errormsg => {
                    this.setHeaderProperty(headerProperty.name, "(nicht bekannt)");
                    console.warn("The header property received is incomplete:", errormsg);
                });
            });

            // photos from header getProperties
            if (this.hasPhotos) {
                api.updateProperty(thingId, "photos", urlArray => {
                    this.headerPhotosUrls = urlArray;
                }, errormsg => {
                // this.setHeaderPhotos("fallback.png");
                    this.headerPhotosUrls = [];
                    console.warn("The header property photos is invalid:", errormsg);
                });
            }

            // means of transport
            if (meansOfTransport) {
                this.meansOfTransport = meansOfTransport.label;
            }
            else {
                this.meansOfTransport = "";
            }
        },

        /**
         * setter for title
         * @param {String} value the title to be shown in the template
         * @returns {Void}  -
         */
        setTitle: function (value) {
            this.title = value;
        },

        /**
         * setter for description
         * @param {String} value the description to be shown in the template
         * @returns {Void}  -
         */
        setDescription: function (value) {
            this.description = value;
        },

        /**
         * setter for header property
         * @param {String} name the name of the property
         * @param {String} value the value
         * @returns {Void}  -
         */
        setHeaderProperty: function (name, value) {
            this.headerProperties.forEach(headerProperty => {
                if (headerProperty.name == name) {
                    headerProperty.value = value;
                }
            });
        },

        /**
         * setter for the component key
         * @param {String} value the dynamic changed value from watch hook
         * @returns {Void}  -
         */
        setComponentKey: function (value) {
            this.keyInfo = value + "info";
            this.keyDay = value + "day";
            this.keyWeek = value + "week";
            this.keyYear = value + "year";
        },

        /**
         * setting the width for day, week and year tabs to show the whole diagram
         * @returns {void}  -
         */
        setGfiDiagramWidth: function () {
            const gfiWrapper = document.querySelector(".gfi-detached");

            if (gfiWrapper) {
                gfiWrapper.style.right = "auto";
                gfiWrapper.style.left = "0px";
                gfiWrapper.style.top = "0px";
                gfiWrapper.style.width = "98%";
                gfiWrapper.style.maxWidth = "992px";
            }
            const gfiContent = document.querySelector(".gfi-content");

            if (gfiContent) {
                gfiContent.style.width = "100%";
                gfiContent.style.maxWidth = "none";
            }
        }
    }
};
</script>

<template>
    <div class="trafficCount-gfi">
        <div class="header">
            <div class="properties">
                <b>{{ idLabel }}:</b> {{ title }}<br><br>
                <b>{{ descriptionLabel }}:</b><br>{{ description }}<br><br>
                <span v-for="headerProperty in headerProperties">
                    <b>{{ headerProperty.label }}:</b> {{ headerProperty.value }}<br>
                </span>
            </div>
            <div
                v-if="hasPhotos"
                class="photos"
            >
                <img
                    v-for="srcUrl in headerPhotosUrls"
                    :src="srcUrl"
                    :alt="srcUrl"
                >
            </div>
        </div>

        <div>
            <ul
                class="nav nav-tabs"
                @click="setCurrentTabId"
            >
                <li
                    value="infos"
                    class="nav-item"
                    :class="(currentTabId == 'infos') ? 'active' : ''"
                >
                    <a
                        href="#infos"
                        class="nav-link"
                    >{{ infoLabel }}</a>
                </li>
                <li
                    value="day"
                    class="nav-item"
                    :class="(currentTabId == 'day') ? 'active' : ''"
                >
                    <a
                        href="#day"
                        class="nav-link"
                    >{{ dayLabel }}</a>
                </li>
                <li
                    value="week"
                    class="nav-item"
                    :class="(currentTabId == 'week') ? 'active' : ''"
                >
                    <a
                        href="#week"
                        class="nav-link"
                    >{{ weekLabel }}</a>
                </li>
                <li
                    value="year"
                    class="nav-item"
                    :class="(currentTabId == 'year') ? 'active' : ''"
                >
                    <a
                        href="#year"
                        class="nav-link"
                    >{{ yearLabel }}</a>
                </li>
            </ul>
            <div class="tab-content">
                <div
                    :key="keyInfo"
                    :class="(currentTabId == 'infos') ? 'd-block' : 'd-none'"
                >
                    <TrafficCountInfo
                        v-for="(mot, index) in meansOfTransports"
                        v-if="mot['showOnInfoTab'] !== false"
                        :label="mot['label']"
                        :mot-type="mot['type']"
                        :api="api"
                        :thing-id="propThingId"
                        :means-of-transport="mot['stream']"
                        :mot-day-interval="mot['dayInterval']"
                    />
                </div>
                <div
                    :key="keyDay"
                    :class="(currentTabId == 'day') ? 'd-block' : 'd-none'"
                >
                    <TrafficCountDay
                        v-for="(mot, index) in meansOfTransports"
                        v-if="mot['stream']"
                        :label="mot['label']"
                        :mot-id="mot['id']"
                        :mot-type="mot['type']"
                        :mot-day-interval="mot['dayInterval']"
                        :api="api"
                        :thing-id="propThingId"
                        :means-of-transport="mot['stream']"
                        :day-start-offset="dayStartOffset"
                    />
                </div>

                <div
                    :key="keyWeek"
                    :class="(currentTabId == 'week') ? 'd-block' : 'd-none'"
                >
                    <TrafficCountWeek
                        v-for="(mot, index) in meansOfTransports"
                        v-if="mot['stream']"
                        :label="mot['label']"
                        :mot-id="mot['id']"
                        :mot-type="mot['type']"
                        :api="api"
                        :thing-id="propThingId"
                        :means-of-transport="mot['stream']"
                        :archive-start-date="archiveStartDate"
                        :week-start-offset="weekStartOffset"
                    />
                </div>

                <div
                    :key="keyYear"
                    :class="(currentTabId == 'year') ? 'd-block' : 'd-none'"
                >
                    <TrafficCountYear
                        v-for="(mot, index) in meansOfTransports"
                        v-if="mot['stream']"
                        :label="mot['label']"
                        :mot-id="mot['id']"
                        :mot-type="mot['type']"
                        :api="api"
                        :thing-id="propThingId"
                        :means-of-transport="mot['stream']"
                        :archive-start-date="archiveStartDate"
                    />
                </div>
            </div>
        </div>

        <br><br>
        <TrafficCountFooter
            class="footer"
            :api="api"
            :thing-id="propThingId"
            :means-of-transport="meansOfTransports[0]['stream']"
            :mot-day-interval="meansOfTransports[0]['dayInterval']"
        />
    </div>
</template>

<style lang="scss" scoped>
.trafficCount-gfi {
    padding: 10px 15px 0 15px;
    @media (max-width: 600px) {
        width: inherit;
        height: inherit;
        padding-left: 10px;
        padding-right: 10px;
        div.graph {
            width: inherit;
            height: inherit;
        }
    }
    .header {
        margin: 0 0 30px 0;
        padding: 0;
        display: flex;
        justify-content: space-between;
        .properties {
          box-sizing: border-box;
          margin-right: 16px;
        }
        .photos {
          width: 35%;
          box-sizing: border-box;
          img {
            max-width: 100%;
          }
        }
    }
    .tab-content {
      border: 1px solid grey;
      padding: 0 5px;
    }
    ul.nav-tabs {
      li {
        margin: 0;
        border-top: 1px solid grey;
        border-right: 1px solid grey;
        a {
          padding: 0.5rem 1rem;
        }
        a:hover {
          background-color: #eee;
        }
      }
      li:first-child {
        border-left: 1px solid grey;
      }
      li.active {
        a {
          color: #fff;
          background-color: #337ab7;
        }
      }
    }
    .footer {
        position: relative;
        display: block;
        width: 100%;
        margin-bottom: 20px
    }
}
</style>
