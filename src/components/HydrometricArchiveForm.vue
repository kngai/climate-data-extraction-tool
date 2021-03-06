<template>
  <div class="container">
    <div class="row">
      <main role="main" property="mainContentOfPage" class="col-md-9 col-md-push-3">
        <h1>{{ currentRouteTitle }}</h1>

        <p>{{ introDatasetText.station.instructions }}</p>
        <p>
          <strong>{{ introDatasetText.station.tipTitle }}</strong>
          <ul>
            <li
              v-for="(pointText, index) in introDatasetText.station.tipPoints"
              :key="index">{{ pointText }}</li>
          </ul>
        </p>

        <data-access-doc-link></data-access-doc-link>

        <details :open="toggleDetailsState">
          <summary @click="toggleDetails"
            v-translate>Dataset description, technical information and metadata</summary>
          <p v-translate>Historical hydrometric data are standardized water resource data and information. They are collected, interpreted and disseminated by the Water Survey of Canada (WSC) in partnership with the provinces, territories and other agencies through the National Hydrometric Program. These data sets include daily mean, monthly mean, annual maximum and minimum daily mean and instantaneous peak water level and discharge information for over 2700 active and 5080 discontinued hydrometric monitoring stations across Canada.</p>

          <p v-html="techDocHtml"></p>

          <p v-html="openPortalHtml"></p>

          <station-list-link
            :url-station-list="urlStationList"
            :download-text="$gettext('Download a list of detailed information for each Historical hydrometric station.')"></station-list-link>
        </details>

        <info-contact-support></info-contact-support>

        <bbox-map
          v-model="ows_bbox"
          :max-zoom="mapMaxZoom"
          :readable-columns="popup_props_display"
          :select-disabled="provinceSelected"
          :geojson="hydroStationsGeoJson"
          :stn-primary-id="stnPrimaryId"
          :hydro-station-display="true"></bbox-map>

        <province-select
          v-model="wfs_province"></province-select>

        <station-select
          v-model="wfs_selected_station_ids"
          :select-disabled="provinceSelected"
          :station-data="hydroStationsGeoJson.features"
          :station-prop-display="station_props_display"
          :station-prov-col="stationProvCol"
          :no-province-station-selected="noProvinceStationSelected"
          :stn-primary-id="stnPrimaryId"
          :hydro-station-display="true"></station-select>

        <var-select
          class="mrgn-tp-md"
          v-model="wfs_layer"
          :label="$gettext('Value type / Time interval')"
          :required="true"
          :select-options="layer_options"></var-select>

        <fieldset>
          <legend v-translate>Date range</legend>
          <date-select
            v-model="date_start"
            :label="$gettext('Start date')"
            :min-date="date_min"
            :max-date="date_max"
            :minimum-view="dateConfigs.minimumView"
            :format="dateConfigs.format"
            :custom-error-msg="dateRangeErrorMessage"
            :placeholder="dateConfigs.placeholder"></date-select>

          <date-select
            v-model="date_end"
            :label="$gettext('End date')"
            :min-date="date_min"
            :max-date="date_max"
            :minimum-view="dateConfigs.minimumView"
            :format="dateConfigs.format"
            :custom-error-msg="dateRangeErrorMessage"
            :placeholder="dateConfigs.placeholder"></date-select>

          <button
            class="btn btn-default"
            type="button"
            @click="clearDates">Clear dates</button>
        </fieldset>

        <format-select-vector
          class="mrgn-tp-md"
          v-model="wfs_format"></format-select-vector>

        <url-box
          :layer-options="selectedLayerOption"
          :ows-url-formatter="wfs3_download_url"
          :wfs3-common-url="getWFS3CommonURL(wfs_layer)"
          :wfs3-download-limit="wfs_limit"
          :layer-format="wfs_format"
          :has-errors="hasErrors"
          :url-box-title="$gettext('Data download links')">
        </url-box>
      </main>
      <dataset-menu></dataset-menu>
    </div>
  </div>
</template>

<script>
import DatasetMenu from './DatasetMenu'
import VarSelect from './VarSelect'
import BBOXMap from './BBOXMap'
import ProvinceSelect from './ProvinceSelect'
import StationSelect from './StationSelect'
import FormatSelectVector from './FormatSelectVector'
import DateSelect from './DateSelect'
import URLBox from './URLBox'
import InfoContactSupport from './InfoContactSupport'
import StationListLink from './StationListLink'
import DataAccessDocLink from './DataAccessDocLink'
import { wfs } from './mixins/wfs'
import { ows } from './mixins/ows'
import { datasets } from './mixins/datasets'

export default {
  name: 'HydrometricArchiveForm',
  mixins: [wfs, ows, datasets],
  components: {
    'dataset-menu': DatasetMenu,
    'bbox-map': BBOXMap,
    'province-select': ProvinceSelect,
    'station-select': StationSelect,
    'format-select-vector': FormatSelectVector,
    'date-select': DateSelect,
    'var-select': VarSelect,
    'url-box': URLBox,
    'info-contact-support': InfoContactSupport,
    'station-list-link': StationListLink,
    DataAccessDocLink
  },
  data () {
    return {
      wfs_layer: 'hydrometric-daily-mean',
      wfs_layer_station: 'hydrometric-stations',
      date_start: this.$moment.utc('1850-01-01', 'YYYY-MM-DD').toDate(),
      date_end: this.$moment.utc().toDate(),
      date_min: this.$moment.utc('1850-01-01', 'YYYY-MM-DD').toDate(),
      date_max: this.$moment.utc().toDate()
    }
  },
  watch: {
    wfs_province: function (newVal, oldVal) {
      this.$store.dispatch('changeProvince', newVal) // to share with bbox
    },
    ows_bbox: function (newVal, oldVal) {
      this.$store.dispatch('changeBBOX', newVal) // to share with station select table
    },
    activeStationsOnly: function (newVal, oldVal) { // display inactive stations
      if (newVal === false) {
        this.$store.dispatch('retrieveHydroStations', this.urlStationList)
      }
    }
  },
  beforeMount () {
    // Load hydrometric stations
    if (this.hydroStationsGeoJson.features.length === 0) { // prevent duplicate AJAX
      this.$store.dispatch('retrieveHydroStations', this.urlStationList)
    }
  },
  computed: {
    activeStationsOnly: function () {
      return this.$store.getters.getHydroStationActive
    },
    urlStationList: function () {
      let url = this.wfs3_url_base + '/' + this.wfs_layer_station + '/items?f=json&limit=' + this.wfs_station_limit +
        `&properties=${this.stationProvCol},${this.datasetToNameColName[this.$route.name]},${this.datasetToStnColName[this.$route.name]},STATUS_EN`
      if (this.activeStationsOnly) {
        url += '&STATUS_EN=Active'
      }
      return url
    },
    hydroStationsGeoJson: function () {
      let hs = this.$store.getters.getHydroStations
      if (hs === null) {
        return null
      }
      let hydroStations = Object.assign({}, hs) // Clone object to prevent original alteration
      if (this.activeStationsOnly) { // filter here so it works with map and table
        if (hydroStations.hasOwnProperty('features')) {
          hydroStations.features = hydroStations.features.filter((feature) => {
            return feature.properties.STATUS_EN === 'Active'
          })
        }
      }
      return hydroStations
    },
    layer_options: function () {
      return {
        'hydrometric-daily-mean': this.$gettext('Daily mean'),
        'hydrometric-monthly-mean': this.$gettext('Monthly mean'),
        'hydrometric-annual-peaks': this.$gettext('Annual max/min'),
        'hydrometric-annual-statistics': this.$gettext('Daily max/min')
      }
    },
    selectedLayerOption: function () {
      var selLayer = {}
      selLayer[this.wfs_layer] = this.layer_options[this.wfs_layer]
      return selLayer
    },
    station_props_display: function () {
      var props = {}
      props[this.datasetToNameColName[this.$route.name]] = this.$gettext('Station name')
      props[this.datasetToStnColName[this.$route.name]] = this.$gettext('Station ID')
      props[this.datasetToProvColName[this.$route.name]] = this.$gettext('Province/Territory/State')
      return props
    },
    popup_props_display: function () {
      var stationCols = Object.keys(this.station_props_display)
      return {
        name: {
          col: stationCols[0],
          label: this.station_props_display[stationCols[0]] + this.$pgettext('Colon', ':')
        },
        id: {
          col: stationCols[1],
          label: this.station_props_display[stationCols[1]] + this.$pgettext('Colon', ':')
        },
        prov: {
          col: stationCols[2],
          label: this.station_props_display[stationCols[2]] + this.$pgettext('Colon', ':')
        }
      }
    },
    dateConfigs: function () {
      if (this.wfs_layer === 'hydrometric-monthly-mean') {
        return {
          minimumView: 'month',
          format: 'YYYY-MM',
          placeholder: 'YYYY-MM'
        }
      } else {
        return {
          minimumView: 'day',
          format: 'YYYY-MM-DD',
          placeholder: 'YYYY-MM-DD'
        }
      }
    },
    temporal: function () {
      if (this.dateRangeIsValid) {
        var start = this.$moment.utc(this.date_start).format(this.dateConfigs.format)
        var end = this.$moment.utc(this.date_end).format(this.dateConfigs.format)
        return 'datetime=' + start + '/' + end
      } else {
        return null
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
</style>
