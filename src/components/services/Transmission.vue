<template>
    <Generic :item="item">
        <template #content>
            <p class="title is-4">{{ item.name }}</p>
            <p class="subtitle is-6">
               <span v-if="serverError" class="error">An error has occurred.</span>
               <template v-else>
                    <span class="down monospace">
                        <i class="fas fa-download" aria-hidden="true"></i>
                        {{ downRate }}
                    </span>
                    <span class="up monospace">
                        <i class="fas fa-upload" aria-hidden="true"></i>
                        {{ upRate }}
                    </span>
                </template>
            </p>
        </template>
        <template #indicator>
            <div class="notifs">
                <strong class="notif downloading" title="Torrents Downloading">
                    {{ downloading }}
                </strong>
                <strong class="notif seeding" title="Torrents Seeding">
                    {{ seeding }}
                </strong>
                <strong class="notif total" title="Total Torrents">
                    {{ total }}
                </strong>
                <strong v-if="serverError" class="notif error"
                    title="Connection error to transmission, check endpoint, username and password in config.yml">?</strong>
            </div>
        </template>
    </Generic>
</template>
  
<script>
import service from "@/mixins/service.js";
import Generic from "./Generic.vue";

const units = ["B", "KB", "MB", "GB"];

// Take the rate in bytes and keep dividing it by 1k until the lowest
// value for which we have a unit is determined. Return the value with
// up to two decimals as a string and unit/s appended.
const displayRate = (rate) => {
    let i = 0;

    while (rate > 1000 && i < units.length) {
        rate /= 1000;
        i++;
    }
    return (
        Intl.NumberFormat(undefined, { maximumFractionDigits: 2 }).format(
            rate || 0
        ) + ` ${units[i]}/s`
    );
};

export default {
    name: "Transmission",
    mixins: [service],
    props: { item: Object },
    components: { Generic },
    data: () => ({ downloading: null, seeding: null, dl: null, ul: null, total: null, serverError: false }),
    computed: {
        downRate: function () {
            return displayRate(this.dl);
        },
        upRate: function () {
            return displayRate(this.ul);
        },
    },
    created() {
        const updateInterval = parseInt(this.item?.updateInterval, 10) || 0;
        if (updateInterval > 0) {
            this.interval = setInterval(() => this.fetchData(), updateInterval);
        }

        this.fetchData();
    },
    unmounted() {
        if (this.interval) {
            clearInterval(this.interval);
        }                   
    },
    methods: {
        fetchData: async function () {
            await this.fetchTransmissionData( `{"method": "session-stats"}`)
                .then((rs) => {
                    this.dl = rs.arguments.downloadSpeed;
                    this.ul = rs.arguments.uploadSpeed;
                    this.total = rs.arguments.torrentCount;
                });
            await this.fetchTransmissionData(`{"method": "torrent-get" , "arguments": {"fields": ["status"]} }`)
                .then((rs) => {
                    let downloading = 0;
                    let seeding = 0;
                    for (let i = 0; i < rs.arguments.torrents.length; i++) {
                        if (rs.arguments.torrents[i].status === 4)
                            downloading++;
                        else if (rs.arguments.torrents[i].status === 6)
                            seeding++;
                    }
                    this.downloading = downloading;
                    this.seeding = seeding;
                });

        },
        fetchTransmissionData: async function (method) {
            const options = {
                method: "POST",
                headers: {
                    "Content-Type": "application/json",
                    "Authorization": `Basic ${btoa(`${this.item.username}:${this.item.password}`)}`
                },
                body: `${method}`,
            };

            if (this.sessionId)
                options.headers["X-Transmission-Session-Id"] = this.sessionId;

            try {
                let response = await this.fetch("/transmission/rpc", options, false, true);

                if (response.status === 409) {
                    this.sessionId = response.headers.get("X-Transmission-Session-Id");
                    return await this.fetchData();
                }

                if (!response.ok) {
                    throw new Error("Not 2xx response")
                }
                return await response.json();
            }
            catch (e) {
                this.serverError = true;
                console.log(e);
            };
        },
    },
};
</script>
  
<style lang="scss" scoped>

.down {
    margin-right: 1em;
}

.count {
    color: var(--text);
    font-size: 0.8em;
}

.monospace {
    font-weight: 300;
    font-family: monospace;
}

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

        &.total {
            background-color: #4fb5d6;
        }

        &.downloading {
            background-color: #4fd671;
        }

        &.seeding {
            background-color: #d08d2e;
        }

        &.error {
            background-color: #e51111;
        }
    }
}
</style>