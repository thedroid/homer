<template>
  <Generic :item="item">
    <template #indicator>
      <div class="notifs">
        <strong v-if="queued > 0" class="notif queued" title="Queued">
          {{ queued }}
        </strong>
        <strong v-if="wanted > 0" class="notif wanted" title="Wanted">
          {{ wanted }}
        </strong>
        <strong v-if="series > 0" class="notif series" title="Series">
          {{ series }}
        </strong>
        <strong
          v-if="serverError"
          class="notif errors"
          title="Connection error to Sonarr API, check your url and apikey in config.yml"
        >
          ?
        </strong>
      </div>
    </template>
  </Generic>
</template>

<script>
import service from "@/mixins/service.js";
import Generic from "./Generic.vue";

const V3_API = "/api/v3";
const LEGACY_API = "/api";

export default {
  name: "Sonarr",
  mixins: [service],
  props: {
    item: Object,
  },
  components: {
    Generic,
  },
  computed: {
    apiPath() {
      return this.item.legacyApi ? LEGACY_API : V3_API;
    },
  },
  data: () => {
    return {
      queued: null,
      wanted: null,
      series: null,
      serverError: false,
    };
  },
  created: function () {
    const updateInterval = parseInt(this.item.updateInterval, 10) || 0;
    if (updateInterval > 0) {
      setInterval(() => this.fetchConfig(), updateInterval);
    }
    this.fetchConfig();
  },
  methods: {
    fetchConfig: function () {
      this.fetch(`${this.apiPath}/queue?apikey=${this.item.apikey}`)
        .then((queue) => {
          this.queued = 0;
          if (this.item.legacyApi) {
            for (var i = 0; i < queue.length; i++) {
              if (queue[i].series) {
                this.queued++;
              }
            }
          } else {
            this.queued = queue.totalRecords;
          }
        })
        .catch((e) => {
          console.error(e);
          this.serverError = true;
        });
      this.fetch(`${this.apiPath}/wanted/missing?apikey=${this.item.apikey}`)
        .then((missing) => {
          this.wanted = missing.totalRecords;
        })
        .catch((e) => {
          console.error(e);
          this.serverError = true;
        });
      this.fetch(`${this.apiPath}/series?apikey=${this.item.apikey}&includeSeasonImages=false`)
        .then((allseries) => {
          this.series = allseries.length;
        })
        .catch((e) => {
          console.error(e);
          this.serverError = true;
        });
    },
  },
};
</script>

<style scoped lang="scss">
.notifs {
  position: absolute;
  color: white;
  font-family: sans-serif;
  top: 0.3em;
  right: 0.5em;

  .notif {
    display: inline-block;
    padding: 0.2em 0.35em;
    border-radius: 0.25em;
    position: relative;
    margin-left: 0.3em;
    font-size: 0.8em;

    &.queued {
      background-color: #4fb5d6;
    }

    &.wanted {
      background-color: #d08d2e;
    }

    &.series {
      background-color: #8dd475;
    }
  }
}
</style>