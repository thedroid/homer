<template>
  <Generic :item="item">
    <template #indicator>
      <div class="notifs">
        <strong v-if="queue > 0" class="notif queue" title="Queued">
          {{ queue }}
        </strong>
        <strong v-if="wanted > 0" class="notif wanted" title="Wanted">
          {{ wanted }}
        </strong>
        <strong v-if="missing > 0" class="notif missing" title="Missing">
          {{ missing }}
        </strong>
        <strong v-if="movie > 0" class="notif movie" title="Movies">
          {{ movie }}
        </strong>
        <strong
          v-if="serverError"
          class="notif errors"
          title="Connection error to Radarr API, check url and apikey in config.yml"
          >?</strong
        >
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
  name: "Radarr",
  mixins: [service],
  props: {
    item: Object,
  },
  components: {
    Generic,
  },
  data: () => {
    return {
      queue: null,
      wanted: null,
      missing: null,
      movie: null,
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
  computed: {
    apiPath() {
      return this.item.legacyApi ? LEGACY_API : V3_API;
    },
  },
  methods: {
    fetchConfig: function () {
      this.fetch(`${this.apiPath}/queue/status?apikey=${this.item.apikey}`)
        .then((queues) => {
          this.queue = queues.totalCount;
        })
        .catch((e) => {
          console.error(e);
          this.serverError = true;
        });
      this.fetch(`${this.apiPath}/movie?apikey=${this.item.apikey}`)
        .then((movies) => {
          this.wanted = 0;
          this.movie = 0;
          this.missing = 0;
          for (var i = 0; i < movies.length;i++) {
                if (movies[i].monitored && !movies[i].hasFile && movies[i].isAvailable)
                   this.wanted++;
                if (movies[i].hasFile)
                   this.movie++;
                if (movies[i].monitored && !movies[i].hasFile)
                   this.missing++;
          }
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
    &.queue {
      background-color: #4fb5d6;
    }

    &.wanted {
      background-color: #d08d2e;
    }

    &.missing {
      background-color: #e51111;
    }

    &.movie {
      background-color: #8dd475;
    }
  }
}
</style>