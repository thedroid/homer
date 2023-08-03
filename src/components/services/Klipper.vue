<template>
  <Generic :item="item" :title="state">
    <template #content>
      <p class="title is-4">{{ item.name }}</p>
      <p class="subtitle is-6">
        <template v-if="item.subtitle && !statusClass">
          {{ item.subtitle }}
        </template>
        <template v-if="statusClass == 'in-progress'">
          <i class="fa-solid fa-gear mr-1"></i>
          <b v-if="display_status.progress">{{ (display_status.progress * 100).toFixed() }}%</b>
          <span class="separator mx-1"> | </span>
          <span v-if="this.file_metadata" :title="`${getETA}`">
            <i class="fa-solid fa-stopwatch mr-1"></i>
            {{ getETA }}
          </span>
        </template>
        <span v-if="error" :title="error">{{ error }}</span>
      </p>
    </template>
    <template #indicator>
      <i :class="['status', statusClass]" :title="state"></i>
      <div class="notifs">
        <strong v-if="error" class="notif errors"
          title="`{{ this.error }}`">?
        </strong>
      </div>
    </template>
  </Generic>
</template>

<script>
import service from "@/mixins/service.js";
import Generic from "./Generic.vue";

export default {
  name: "Klipper",
  mixins: [service],
  props: {
    item: Object,
  },
  components: {
    Generic,
  },
  data: () => ({
    display_status: null,
    print_status: null,
    webhooks: null,
    file_metadata: null,
    virtual_sdcard: null,
    error: null,
  }),
  computed: {
    statusClass: function () {
      switch (this.print_status?.state) {
        case "standby":
          return "ready";
        case "paused":
          return "Paused";
        case "printing":
          return "in-progress";
        case "error":
          return "error";
        default:
          return "pending";
      }
    },
    getETA: function () {
      const etas = [];

      const { total_duration, filament_used } = this.print_status;
      const { size, gcode_start_byte, gcode_end_byte, filament_total, estimated_time } = this.file_metadata;
      const { file_position } = this.virtual_sdcard;

      // By slicer ETA if it was provided.
      const totalVsActual = total_duration - this.print_status.print_duration;

      if (estimated_time)
        etas.push(estimated_time + totalVsActual - total_duration);

      // If slicer ETA was not provided use ETA by Filesize.  If Slicer ETA was provided
      // only start average out if we are over 10% complete.
      if (!estimated_time || this.display_status.progress > .10) {
        // By File Size
        const effectiveFileSize = size - (gcode_start_byte + (size - gcode_end_byte));
        const remainingBytes = effectiveFileSize - file_position;
        const averageTimePerByte = total_duration / file_position;
        etas.push(averageTimePerByte * remainingBytes);
      }

      // only start average out if we are over 10% complete.
      if (this.display_status.progress > .10) {
        // By Filament Usage
        const filamentRemaining = filament_total - (filament_used + 0.1);
        const averageSpeed = (filament_used + 0.1) / total_duration;
        etas.push(filamentRemaining / averageSpeed);
      }
      //Average it Out
      const avg = etas.reduce((acc, val) => acc + val, 0) / etas.length;
      const s = (Math.floor(avg / 86400) + ":" + (new Date(avg * 1000)).toISOString().substr(11, 8)).split(":");

      return `${s[0]}:${s[1]}:${s[2]}:${s[3]}`;
    },
  },
  created() {
    const updateInterval = parseInt(this.item.updateInterval, 10) || 0;
    if (updateInterval > 0) {
      setInterval(() => this.fetchStatus(), updateInterval);
    }
    this.fetchStatus();


    this.fetchStatus();
  },
  methods: {
    fetchStatus: async function () {
      try {
        await this.fetch("/printer/objects/query?display_status&webhooks&print_stats&virtual_sdcard")
          .then((r) => {
            if (r.result.status) {
              this.webhooks = r.result.status.webhooks;
              if (r.result.status.webhooks.state === "ready") {

                let oldfilename = null;

                if (this.print_status)
                  oldfilename = this.print_status.filename;

                this.display_status = r.result.status.display_status;
                this.print_status = r.result.status.print_stats;
                this.virtual_sdcard = r.result.status.virtual_sdcard;

                if (this.print_status.state === "printing") {
                  if (!oldfilename || oldfilename != this.print_status.filename) {
                    this.fetch(`/server/files/metadata?filename=${this.print_status.filename}`)
                      .then((r) => {
                        this.file_metadata = r.result;
                      })
                  }
                }
              }
            }
          });
      } catch (e) {
        this.error = `Fail to fetch moonraker (${e.message})`;
        console.error(e);
      }
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

    &.errors {
      background-color: #e51111;
    }
  }
}
</style>