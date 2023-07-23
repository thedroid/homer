<template>
    <Generic :item="item">
      <template #content>
        <p class="title is-4">{{ item.name }}</p>
        <p class="subtitle is-6">
          <span v-if="serverError" class="error">An error has occurred.</span>
          <template v-else>
            <span>{{ location }}</span>
          </template>
        </p>
      </template>
      <template #indicator>
        <span v-if="!serverError" class="ip"
          >{{ ip }}
        </span>
      </template>
    </Generic>
  </template>
  
  <script>
  import service from "@/mixins/service.js";
  import Generic from "./Generic.vue";
  
  export default {
    name: "GeoIP",
    mixins: [service],
    props: {
      item: Object,
    },
    components: {
      Generic,
    },
    data: () => ({
      waninfo: null,
      serverError: false,
    }),
    computed: {
      ip: function () {
        return this.waninfo?.ip;
      },
      location: function () {
        return this.waninfo?.city + ", " + this.waninfo?.country;
      },
      org: function () {
        return ` (${this.waninfo?.asn_org})`;
      },
      lat: function () {
        return ` (${this.waninfo?.latitude})`;
      },
      long: function () {
        return ` (${this.waninfo?.longitude})`;
      },
      timezone: function () {
        return ` (${this.waninfo?.time_zone})`;
      },
      country: function () {
        return ` (${this.waninfo?.country_iso})`;
      },
    },
    created() {
      this.fetchStatus();
    },
    methods: {
      fetchStatus: async function () {
        const headers = {
          "Accept": "application/json",
        };
        this.fetch("/", { headers })
        .then((response) => {
          this.waninfo = response;
       })
        .catch((e) => {
          console.log(e);
          this.serverError = true;
        });
      },
    },
  };
  </script>
  
  <style scoped lang="scss">
  .ip {
    font-size: 0.8rem;
    color: var(--text-title);
    white-space: nowrap;
    margin-left: 0.25rem;
  }
  </style>