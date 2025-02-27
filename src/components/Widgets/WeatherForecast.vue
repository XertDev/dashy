<template>
<div class="weather-forecast">
  <template v-if="weatherData.length > 0">
    <!-- For each day, show the weather -->
    <div
      class="weather-day"
      v-for="weather in weatherData"
      :key="weather.index"
      v-tooltip="tooltip(weather.description)"
      @click="showMoreInfo(weather.info)"
    >
      <p class="date">{{ weather.date }}</p>
      <p class="description">{{ weather.main }}</p>
      <p class="temp">{{ weather.temp }}</p>
      <i :class="`owi owi-${weather.icon}`"></i>
    </div>
  </template>
  <!-- Show more details for a Clicked day -->
  <div class="details" v-if="showDetails">
    <div class="info-wrap" v-for="(section, indx) in moreInfo" :key="indx">
      <p class="info-line" v-for="weather in section" :key="weather.label">
        <span class="lbl">{{weather.label}}</span>
        <span class="val">{{ weather.value }}</span>
      </p>
    </div>
  </div>
  <p class="more-details-btn" @click="toggleDetails" v-if="showDetails">
    {{ $t('widgets.general.show-less') }}
  </p>
</div>
</template>

<script>
import axios from 'axios';
import WidgetMixin from '@/mixins/WidgetMixin';
import { widgetApiEndpoints } from '@/utils/defaults';

export default {
  mixins: [WidgetMixin],
  data() {
    return {
      loading: true,
      showDetails: false,
      weatherData: [],
      moreInfo: [],
    };
  },
  computed: {
    units() {
      return this.options.units || 'metric';
    },
    numDays() {
      return this.options.numDays || 6;
    },
    endpoint() {
      const { apiKey, city } = this.options;
      const params = `?q=${city}&cnt=${this.numDays}&units=${this.units}&appid=${apiKey}`;
      return `${widgetApiEndpoints.weatherForecast}${params}`;
    },
    tempDisplayUnits() {
      switch (this.units) {
        case ('metric'): return '°C';
        case ('imperial'): return '°F';
        default: return '';
      }
    },
    speedDisplayUnits() {
      switch (this.units) {
        case ('metric'): return 'm/s';
        case ('imperial'): return 'mph';
        default: return '';
      }
    },
  },
  methods: {
    /* Extends mixin, and updates data. Called by parent component */
    update() {
      this.startLoading();
      this.fetchWeather();
    },
    /* Adds units symbol to temperature, depending on metric or imperial */
    processTemp(temp) {
      return `${Math.round(temp)}${this.tempDisplayUnits}`;
    },
    /* Convert timestamp to textual date, in users local format */
    dateFromStamp(timestamp) {
      const localFormat = navigator.language;
      const dateFormat = { weekday: 'short', day: 'numeric', month: 'short' };
      return new Date(timestamp * 1000).toLocaleDateString(localFormat, dateFormat);
    },
    /* Fetches the weather from OpenWeatherMap, and processes results */
    fetchWeather() {
      axios.get(this.endpoint)
        .then((response) => {
          if (response.data.list) {
            this.processApiResults(response.data);
          }
        })
        .catch((error) => {
          this.error('Failed to fetch weather', error);
        })
        .finally(() => {
          this.finishLoading();
        });
    },
    /* Process the results from the Axios request */
    processApiResults(dataList) {
      const uiWeatherData = [];
      dataList.list.forEach((day, index) => {
        uiWeatherData.push({
          index,
          date: this.dateFromStamp(day.dt),
          icon: day.weather[0].icon,
          main: day.weather[0].main,
          description: day.weather[0].description,
          temp: this.processTemp(day.temp.day),
          info: this.makeWeatherData(day),
        });
      });
      this.weatherData = uiWeatherData;
    },
    /* Process additional data, needed when user clicks a given day */
    makeWeatherData(data) {
      return [
        [
          { label: 'Min Temp', value: this.processTemp(data.temp.min) },
          { label: 'Max Temp', value: this.processTemp(data.temp.max) },
          { label: 'Feels Like', value: this.processTemp(data.feels_like.day) },
        ],
        [
          { label: 'Pressure', value: `${data.pressure}hPa` },
          { label: 'Humidity', value: `${data.humidity}%` },
          { label: 'wind', value: `${data.speed}${this.speedDisplayUnits}` },
          { label: 'clouds', value: `${data.clouds}%` },
        ],
      ];
    },
    /* When a day is clicked, then show weather info on the UI */
    showMoreInfo(moreInfo) {
      this.moreInfo = moreInfo;
      this.showDetails = true;
    },
    /* Show/ hide additional weather info */
    toggleDetails() {
      this.showDetails = !this.showDetails;
    },
    /* Display weather description and Click for more note on hover */
    tooltip(text) {
      const content = `${text.split(' ').map(
        (word) => word[0].toUpperCase() + word.substring(1),
      ).join(' ')}\nClick for more Info`;
      return { content, trigger: 'hover focus', delay: 250 };
    },
    /* Validate input props, and print warning if incorrect */
    checkProps() {
      const ops = this.options;
      let valid = true;
      if (!ops.apiKey) {
        this.error('Missing API key for OpenWeatherMap');
        valid = false;
      }
      if (!ops.city) {
        this.error('A city name is required to fetch weather');
        valid = false;
      }
      if (ops.units && ops.units !== 'metric' && ops.units !== 'imperial') {
        this.error('Invalid units specified, must be either \'metric\' or \'imperial\'');
        valid = false;
      }
      return valid;
    },
  },
  /* When the widget loads, the props are checked, and weather fetched */
  created() {
    if (this.checkProps()) {
      this.fetchWeather();
    }
  },
};
</script>

<style scoped lang="scss">
@import '@/styles/weather-icons.scss';

  p {
    color: var(--widget-text-color);
  }

.weather-forecast {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  .weather-day {
    display: grid;
    gap: 0.25rem;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    width: 30%;
    min-width: 6rem;
    cursor: pointer;
    text-align: center;
    padding: 0.5rem 0.25rem;
    border: 1px solid transparent;
    border-radius: var(--curve-factor);
    &:hover {
      border: 1px dashed var(--widget-text-color);
    }
    .date {
      grid-column-start: span 2;
      opacity: var(--dimming-factor);
      margin: 0;
      text-decoration: underline;
    }
    .description {
      text-transform: capitalize;
      text-align: center;
      margin: 0;
    }
    .owi {
      font-size: 1.4rem;
      color: var(--widget-text-color);
      margin: 0;
    }
    .temp {
      font-size: 1.8rem;
      grid-row-start: span 2;
      display: flex;
      align-items: center;
      justify-content: center;
      margin: 0;
    }
  }

  // Show more details button
  .more-details-btn {
    grid-column-start: span 2;
    cursor: pointer;
    font-size: 0.9rem;
    text-align: center;
    width: fit-content;
    margin: 0.25rem auto;
    padding: 0.1rem 0.25rem;
    border: 1px solid transparent;
    opacity: var(--dimming-factor);
    border-radius: var(--curve-factor);
    &:hover {
      border: 1px solid var(--widget-text-color);
    }
    &:focus, &:active {
      background: var(--widget-text-color);
      color: var(--widget-background-color);
    }
  }
  // More weather details table
  .details {
    width: 100%;
    grid-column-start: span 2;
    display: flex;
    .info-wrap {
      display: flex;
      flex-direction: column;
      width: 100%;
      opacity: var(--dimming-factor);
      p.info-line {
        display: flex;
        justify-content: space-between;
        margin: 0.1rem 0.5rem;
        padding: 0.1rem 0;
        color: var(--widget-text-color);
        &:not(:last-child) {
          border-bottom: 1px dashed var(--widget-text-color);
        }
      }
    }
  }
}

</style>
