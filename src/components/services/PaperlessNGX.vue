<template>
    <Generic :item="item">
      <template #indicator>
        <div class="notifs">
          <strong v-if="documentcount > 0" class="notif documentcount" title="Documents">
            {{ documentcount }}
          </strong>
          <strong v-if="inboxcount > 0" class="notif inboxcount" title="Inbox">
            {{ inboxcount }}
          </strong>
          <strong
            v-if="serverError"
            class="notif errors"
            title="Connection error to PaperlessNGX API, check url and apikey in config.yml"
            >?</strong
          >
        </div>
      </template>
    </Generic>
  </template>
  
  <script>
  import service from "@/mixins/service.js";
  import Generic from "./Generic.vue";
  
  export default {
    name: "PaperlessNGX",
    mixins: [service],
    props: {
      item: Object,
    },
    components: {
      Generic,
    },
    data: () => {
      return {
        documentcount: null,
        inboxcount: null,
        serverError: false,
      };
    },
    created() {
      this.fetchStatus();
    },
    methods: {
      fetchStatus: async function () {
  
        const apikey = this.item.apikey;
        if (!apikey) {
          this.serverError = true;
          console.error(
            "Apikey is not present!"
          );
          return;
        }
        this.fetch("/api/statistics/", {
          headers: {
            Authorization: "Token " + this.item.apikey,
          },
        })
        .then((response) => {
          this.documentcount=0;
          this.inboxcount=0;
          this.documentcount = response.documents_total;
          this.inboxcount = response.documents_inbox;
        })
        .catch((e) => console.log(e));
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
  
      &.documentcount {
        background-color: #4fb5d6;
      }
  
      &.inboxcount {
        background-color: #4fd671;
      }
  
      &.errors {
        background-color: #e51111;
      }
    }
  }
  </style>
