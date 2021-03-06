<!--
  - Copyright 2014-2018 the original author or authors.
  -
  - Licensed under the Apache License, Version 2.0 (the "License");
  - you may not use this file except in compliance with the License.
  - You may obtain a copy of the License at
  -
  -     http://www.apache.org/licenses/LICENSE-2.0
  -
  - Unless required by applicable law or agreed to in writing, software
  - distributed under the License is distributed on an "AS IS" BASIS,
  - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  - See the License for the specific language governing permissions and
  - limitations under the License.
  -->

<template>
  <table class="metrics table is-fullwidth">
    <thead>
      <tr>
        <th class="metrics__label" v-text="metricName"/>
        <th class="metrics__statistic-name"
            v-for="statistic in statistics"
            :key="`head-${statistic}`">
          <span v-text="statistic"/>
          <div class="select is-small is-pulled-right">
            <select :value="statisticTypes[statistic]"
                    @change="$emit('type-select', metricName, statistic, $event.target.value)">
              <option :value="undefined">-</option>
              <option value="integer">Integer</option>
              <option value="float">Float</option>
              <option value="duration">Duration</option>
              <option value="millis">Milliseconds</option>
              <option value="bytes">Bytes</option>
            </select>
          </div>
        </th>
        <td/>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(tags, idx) in tagSelections" :key="idx">
        <td class="metrics__label">
          <span v-text="getLabel(tags)"/>
          <span class="has-text-warning" v-if="errors[idx]" :title="errors[idx]">
            <font-awesome-icon icon="exclamation-triangle"/>
          </span>
        </td>
        <td class="metrics__statistic-value"
            v-for="statistic in statistics"
            :key="`value-${idx}-${statistic}`"
            v-text="getValue(measurements[idx], statistic)"
        />
        <td class="metrics__actions">
          <sba-icon-button :icon="'trash'" @click.stop="handleRemove(idx)"/>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script>
  import subscribing from '@/mixins/subscribing';
  import Instance from '@/services/instance';
  import {Observable} from '@/utils/rxjs';
  import _ from 'lodash';
  import moment from 'moment';
  import prettyBytes from 'pretty-bytes';

  const formatDuration = value => {
    const duration = moment.duration(value * 1000);
    return `${Math.floor(duration.asDays())}d ${duration.hours()}h ${duration.minutes()}m ${duration.seconds()}s`;
  };

  const formatMillis = value => {
    return `${moment.duration(value * 1000).asMilliseconds().toFixed(0)} ms`;
  };

  export default {
    name: 'Metric',
    mixins: [subscribing],
    props: {
      metricName: {
        type: String,
        required: true
      },
      instance: {
        type: Instance,
        required: true
      },
      tagSelections: {
        type: Array,
        default: () => [{}]
      },
      statisticTypes: {
        type: Object,
        default: () => ({})
      }
    },
    data: () => ({
      measurements: [],
      statistics: [],
      errors: [],
    }),
    methods: {
      handleRemove(idx) {
        this.$emit('remove', this.metricName, idx);
      },
      getValue(measurements, statistic) {
        const measurement = measurements && measurements.find(m => m.statistic === statistic);
        if (!measurement) {
          return undefined;
        }
        const type = this.statisticTypes[statistic];
        switch (type) {
          case 'integer':
            return measurement.value.toFixed(0);
          case 'float':
            return measurement.value.toFixed(4);
          case 'duration':
            return formatDuration(measurement.value);
          case 'millis':
            return formatMillis(measurement.value);
          case 'bytes':
            return prettyBytes(measurement.value);
          default:
            return measurement.value;
        }
      },
      getLabel(tags) {
        return _.entries(tags).filter(([, value]) => typeof value !== 'undefined')
          .map(pair => pair.join(':'))
          .join('\n') || '(no tags)';
      },
      async fetchMetric(tags, idx) {
        try {
          const response = await this.instance.fetchMetric(this.metricName, tags);
          this.$set(this.errors, idx, null);
          this.$set(this.measurements, idx, response.data.measurements);
          if (idx === 0) {
            this.statistics = response.data.measurements.map(m => m.statistic);
          }
        } catch (error) {
          console.warn(`Fetching metric ${this.metricName} failed:`, error);
          this.$set(this.errors, idx, error);
        }
      },
      fetchAllTags() {
        return Observable.from(this.tagSelections).concatMap(this.fetchMetric);
      },
      createSubscription() {
        const vm = this;
        return Observable.timer(0, 2500)
          .concatMap(vm.fetchAllTags)
          .subscribe({
            next: () => {
            }
          });
      }
    },
    watch: {
      tagSelections(newVal, oldVal) {
        newVal.map((v, i) => [v, i])
          .filter(([v]) => !oldVal.includes(v))
          .forEach(([v, i]) => this.fetchMetric(v, i));
      }
    }
  }
</script>
<style lang="scss">

  table .metrics {
    &__label {
      width: 300px;
      white-space: pre-wrap;
    }
    &__actions {
      width: 1px;
      vertical-align: middle;
    }
    &__statistic-name * {
      vertical-align: middle;
    }

    &__statistic-value {
      text-align: right;
      vertical-align: middle;
    }
  }
</style>
