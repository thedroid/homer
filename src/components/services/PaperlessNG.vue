<template>
  <Generic :item="item">
    <template #content>
      <p class="title is-4">{{ item.name }}</p>
      <p class="subtitle is-6">
        <template v-if="item.subtitle !=null">
          {{ item.subtitle }}
        </template>
        <template v-else-if="api">
          Inbox: {{ api.documents_inbox }} , Total Documents: {{ api.documents_total }} 
        </template>
      </p>
    </template>
    <template #indicator>
      <div class="notifs">
        <strong v-if="api && useNotifs && api.documents_inbox >0" class="notif inboxcount" title="Inbox">
          {{ api.documents_inbox }}
        </strong>
        <strong v-if="api && useNotifs && api.documents_total >0" class="notif documentcount" title="Documents">
          {{ api.documents_total }}
        </strong>
        <strong v-if="serverError" class="notif errors"
          title="Connection error to PaperlessNG API, check url and apikey in config.yml">?</strong>
      </div>
    </template>    
  </Generic>
</template>

<script>
import service from "@/mixins/service.js";
import Generic from "./Generic.vue";

export default {
  name: "PaperlessNG",
  mixins: [service],
  props: {
    item: Object,
  },
  components: {
    Generic,
  },
  data: () => ({
    api: null,
    useNotifs: false,
    serverError: false,
  }),
  created() {
    this.fetchStatus();
  },
  methods: {
    fetchStatus: async function () {
      this.useNotifs = (this.item.subtitle != null);

      const apikey = this.item.apikey;
      if (!apikey) {
        this.serverError = true;
        console.error(
          "apikey is not present in config.yml for the paperless entry!"
        );
        return;
      }
      this.api = await this.fetch("/api/statistics/", {
        headers: {
          Authorization: "Token " + this.item.apikey,
        },
      }).catch((e) => {
          this.serverError = true;
          console.log(e)}
      );
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