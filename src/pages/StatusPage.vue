<template>
    <div v-if="loadedTheme" class="container mt-3">
        <!-- Logo & Title -->
        <h1 class="mb-4">
            <!-- Logo -->
            <span class="logo-wrapper" @click="showImageCropUploadMethod">
                <img :src="logoURL" alt class="logo me-2" :class="logoClass" />
                <font-awesome-icon v-if="enableEditMode" class="icon-upload" icon="upload" />
            </span>

            <!-- Uploader -->
            <!--    url="/api/status-page/upload-logo" -->
            <ImageCropUpload v-model="showImageCropUpload"
                             field="img"
                             :width="128"
                             :height="128"
                             :langType="$i18n.locale"
                             img-format="png"
                             :noCircle="true"
                             :noSquare="false"
                             @crop-success="cropSuccess"
            />

            <!-- Title -->
            <Editable v-model="config.title" tag="span" :contenteditable="editMode" :noNL="true" />
        </h1>

        <!-- Admin functions -->
        <div v-if="hasToken" class="mb-4">
            <div v-if="!enableEditMode">
                <button class="btn btn-info me-2" @click="edit">
                    <font-awesome-icon icon="edit" />
                    {{ $t("Edit Status Page") }}
                </button>

                <a href="/dashboard" class="btn btn-info">
                    <font-awesome-icon icon="tachometer-alt" />
                    {{ $t("Go to Dashboard") }}
                </a>
            </div>

            <div v-else>
                <button class="btn btn-success me-2" @click="save">
                    <font-awesome-icon icon="save" />
                    {{ $t("Save") }}
                </button>

                <button class="btn btn-danger me-2" @click="discard">
                    <font-awesome-icon icon="save" />
                    {{ $t("Discard") }}
                </button>

                <button class="btn btn-primary btn-add-group me-2" @click="createIncident">
                    <font-awesome-icon icon="bullhorn" />
                    {{ $t("Create Incident") }}
                </button>

                <!--
                <button v-if="isPublished" class="btn btn-light me-2" @click="">
                    <font-awesome-icon icon="save" />
                    {{ $t("Unpublish") }}
                </button>

                <button v-if="!isPublished" class="btn btn-info me-2" @click="">
                    <font-awesome-icon icon="save" />
                    {{ $t("Publish") }}
                </button>-->

                <!-- Set Default Language -->
                <!-- Set theme -->
                <button v-if="theme == 'dark'" class="btn btn-light me-2" @click="changeTheme('light')">
                    <font-awesome-icon icon="save" />
                    {{ $t("Switch to Light Theme") }}
                </button>

                <button v-if="theme == 'light'" class="btn btn-dark me-2" @click="changeTheme('dark')">
                    <font-awesome-icon icon="save" />
                    {{ $t("Switch to Dark Theme") }}
                </button>
            </div>
        </div>

        <!-- Incident -->
        <div v-if="incident !== null" class="shadow-box alert mb-4 p-4 incident" role="alert" :class="incidentClass">
            <strong v-if="editIncidentMode">{{ $t("Title") }}:</strong>
            <Editable v-model="incident.title" tag="h4" :contenteditable="editIncidentMode" :noNL="true" class="alert-heading" />

            <strong v-if="editIncidentMode">{{ $t("Content") }}:</strong>
            <Editable v-model="incident.content" tag="div" :contenteditable="editIncidentMode" class="content" />

            <!-- Incident Date -->
            <div class="date mt-3">
                Created: {{ incident.createdDate }} ({{ createdDateFromNow }})<br />
                <span v-if="incident.lastUpdatedDate">
                    Last Updated: {{ incident.lastUpdatedDate }} ({{ lastUpdatedDateFromNow }})
                </span>
            </div>

            <div v-if="editMode" class="mt-3">
                <button v-if="editIncidentMode" class="btn btn-light me-2" @click="postIncident">
                    <font-awesome-icon icon="bullhorn" />
                    {{ $t("Post") }}
                </button>

                <button v-if="!editIncidentMode && incident.id" class="btn btn-light me-2" @click="editIncident">
                    <font-awesome-icon icon="edit" />
                    {{ $t("Edit") }}
                </button>

                <button v-if="editIncidentMode" class="btn btn-light me-2" @click="cancelIncident">
                    <font-awesome-icon icon="times" />
                    {{ $t("Cancel") }}
                </button>

                <div v-if="editIncidentMode" class="dropdown d-inline-block me-2">
                    <button id="dropdownMenuButton1" class="btn btn-secondary dropdown-toggle" type="button" data-bs-toggle="dropdown" aria-expanded="false">
                        Style: {{ incident.style }}
                    </button>
                    <ul class="dropdown-menu" aria-labelledby="dropdownMenuButton1">
                        <li><a class="dropdown-item" href="#" @click="incident.style = 'info'">info</a></li>
                        <li><a class="dropdown-item" href="#" @click="incident.style = 'warning'">warning</a></li>
                        <li><a class="dropdown-item" href="#" @click="incident.style = 'danger'">danger</a></li>
                        <li><a class="dropdown-item" href="#" @click="incident.style = 'primary'">primary</a></li>
                        <li><a class="dropdown-item" href="#" @click="incident.style = 'light'">light</a></li>
                        <li><a class="dropdown-item" href="#" @click="incident.style = 'dark'">dark</a></li>
                    </ul>
                </div>

                <button v-if="!editIncidentMode && incident.id" class="btn btn-light me-2" @click="unpinIncident">
                    <font-awesome-icon icon="unlink" />
                    {{ $t("Unpin") }}
                </button>
            </div>
        </div>

        <!-- Overall Status -->
        <div class="shadow-box list  p-4 overall-status mb-4">
            <div v-if="Object.keys($root.publicMonitorList).length === 0 && loadedData">
                <font-awesome-icon icon="question-circle" class="ok" />
                {{ $t("No Services") }}
            </div>

            <template v-else>
                <div v-if="allUp">
                    <font-awesome-icon icon="check-circle" class="ok" />
                    {{ $t("All Systems Operational") }}
                </div>

                <div v-else-if="partialDown">
                    <font-awesome-icon icon="exclamation-circle" class="warning" />
                    {{ $t("Partially Degraded Service") }}
                </div>

                <div v-else-if="allDown">
                    <font-awesome-icon icon="times-circle" class="danger" />
                    {{ $t("Degraded Service") }}
                </div>

                <div v-else>
                    <font-awesome-icon icon="question-circle" style="color: #efefef" />
                </div>
            </template>
        </div>

        <!-- Description -->
        <strong v-if="editMode">{{ $t("Description") }}:</strong>
        <Editable v-model="config.description" :contenteditable="editMode" tag="div" class="mb-4 description" />

        <div v-if="editMode" class="mb-4">
            <div>
                <button class="btn btn-primary btn-add-group me-2" @click="addGroup">
                    <font-awesome-icon icon="plus" />
                    {{ $t("Add Group") }}
                </button>
            </div>

            <div class="mt-3">
                <div v-if="allMonitorList.length > 0 && loadedData">
                    <label>{{ $t("Add a monitor") }}:</label>
                    <select v-model="selectedMonitor" class="form-control">
                        <option v-for="monitor in allMonitorList" :key="monitor.id" :value="monitor">{{ monitor.name }}</option>
                    </select>
                </div>
                <div v-else class="text-center">
                    {{ $t("No monitors available.") }}  <router-link to="/add">{{ $t("Add one") }}</router-link>
                </div>
            </div>
        </div>

        <div class="mb-4">
            <div v-if="$root.publicGroupList.length === 0 && loadedData" class="text-center">
                <!-- 👀 Nothing here, please add a group or a monitor. -->
                👀 {{ $t("statusPageNothing") }}
            </div>

            <PublicGroupList :edit-mode="enableEditMode" />
        </div>

        <footer class="mt-5 mb-4">
            Powered by <a target="_blank" href="https://github.com/louislam/uptime-kuma">Uptime Kuma</a>
        </footer>
    </div>
</template>

<script>
import axios from "axios";
import PublicGroupList from "../components/PublicGroupList.vue";
import ImageCropUpload from "vue-image-crop-upload";
import { STATUS_PAGE_ALL_DOWN, STATUS_PAGE_ALL_UP, STATUS_PAGE_PARTIAL_DOWN, UP } from "../util.ts";
import { useToast } from "vue-toastification";
import dayjs from "dayjs";
const toast = useToast();

const leavePageMsg = "Do you really want to leave? you have unsaved changes!";

let feedInterval;

export default {
    components: {
        PublicGroupList,
        ImageCropUpload
    },

    // Leave Page for vue route change
    beforeRouteLeave(to, from, next) {
        if (this.editMode) {
            const answer = window.confirm(leavePageMsg);
            if (answer) {
                next();
            } else {
                next(false);
            }
        }
        next();
    },

    data() {
        return {
            enableEditMode: false,
            enableEditIncidentMode: false,
            hasToken: false,
            config: {},
            selectedMonitor: null,
            incident: null,
            previousIncident: null,
            showImageCropUpload: false,
            imgDataUrl: "/icon.svg",
            loadedTheme: false,
            loadedData: false,
            baseURL: "",
        };
    },
    computed: {

        logoURL() {
            if (this.imgDataUrl.startsWith("data:")) {
                return this.imgDataUrl;
            } else {
                return this.baseURL + this.imgDataUrl;
            }
        },

        /**
         * If the monitor is added to public list, which will not be in this list.
         */
        allMonitorList() {
            let result = [];

            for (let id in this.$root.monitorList) {
                if (this.$root.monitorList[id] && ! (id in this.$root.publicMonitorList)) {
                    let monitor = this.$root.monitorList[id];
                    result.push(monitor);
                }
            }

            return result;
        },

        editMode() {
            return this.enableEditMode && this.$root.socket.connected;
        },

        editIncidentMode() {
            return this.enableEditIncidentMode;
        },

        isPublished() {
            return this.config.statusPagePublished;
        },

        theme() {
            return this.config.statusPageTheme;
        },

        logoClass() {
            if (this.editMode) {
                return {
                    "edit-mode": true,
                };
            }
            return {};
        },

        incidentClass() {
            return "bg-" + this.incident.style;
        },

        overallStatus() {

            if (Object.keys(this.$root.publicLastHeartbeatList).length === 0) {
                return -1;
            }

            let status = STATUS_PAGE_ALL_UP;
            let hasUp = false;

            for (let id in this.$root.publicLastHeartbeatList) {
                let beat = this.$root.publicLastHeartbeatList[id];

                if (beat.status === UP) {
                    hasUp = true;
                } else {
                    status = STATUS_PAGE_PARTIAL_DOWN;
                }
            }

            if (! hasUp) {
                status = STATUS_PAGE_ALL_DOWN;
            }

            return status;
        },

        allUp() {
            return this.overallStatus === STATUS_PAGE_ALL_UP;
        },

        partialDown() {
            return this.overallStatus === STATUS_PAGE_PARTIAL_DOWN;
        },

        allDown() {
            return this.overallStatus === STATUS_PAGE_ALL_DOWN;
        },

        createdDateFromNow() {
            return dayjs.utc(this.incident.createdDate).fromNow();
        },

        lastUpdatedDateFromNow() {
            return dayjs.utc(this.incident. lastUpdatedDate).fromNow();
        }

    },
    watch: {

        /**
         * Selected a monitor and add to the list.
         */
        selectedMonitor(monitor) {
            if (monitor) {
                if (this.$root.publicGroupList.length === 0) {
                    this.addGroup();
                }

                const firstGroup = this.$root.publicGroupList[0];

                firstGroup.monitorList.push(monitor);
                this.selectedMonitor = null;
            }
        },

        // Set Theme
        "config.statusPageTheme"() {
            this.$root.statusPageTheme = this.config.statusPageTheme;
            this.loadedTheme = true;
        },

        "config.title"(title) {
            document.title = title;
        }

    },
    async created() {
        this.hasToken = ("token" in this.$root.storage());

        // Browser change page
        // https://stackoverflow.com/questions/7317273/warn-user-before-leaving-web-page-with-unsaved-changes
        window.addEventListener("beforeunload", (e) => {
            if (this.editMode) {
                (e || window.event).returnValue = leavePageMsg;
                return leavePageMsg;
            } else {
                return null;
            }
        });

        // Special handle for dev
        const env = process.env.NODE_ENV;
        if (env === "development" || localStorage.dev === "dev") {
            this.baseURL = location.protocol + "//" + location.hostname + ":3001";
        }
    },
    async mounted() {
        axios.get("/api/status-page/config").then((res) => {
            this.config = res.data;

            if (this.config.logo) {
                this.imgDataUrl = this.config.logo;
            }
        });

        axios.get("/api/status-page/incident").then((res) => {
            if (res.data.ok) {
                this.incident = res.data.incident;
            }
        });

        axios.get("/api/status-page/monitor-list").then((res) => {
            this.$root.publicGroupList = res.data;
        });

        // 5mins a loop
        this.updateHeartbeatList();
        feedInterval = setInterval(() => {
            this.updateHeartbeatList();
        }, (300 + 10) * 1000);
    },
    methods: {

        updateHeartbeatList() {
            // If editMode, it will use the data from websocket.
            if (! this.editMode) {
                axios.get("/api/status-page/heartbeat").then((res) => {
                    this.$root.heartbeatList = res.data.heartbeatList;
                    this.$root.uptimeList = res.data.uptimeList;
                    this.loadedData = true;
                });
            }
        },

        edit() {
            this.$root.initSocketIO(true);
            this.enableEditMode = true;
        },

        save() {
            this.$root.getSocket().emit("saveStatusPage", this.config, this.imgDataUrl, this.$root.publicGroupList, (res) => {
                if (res.ok) {
                    this.enableEditMode = false;
                    this.$root.publicGroupList = res.publicGroupList;
                    location.reload();
                } else {
                    toast.error(res.msg);
                }
            });
        },

        monitorSelectorLabel(monitor) {
            return `${monitor.name}`;
        },

        addGroup() {
            let groupName = "Untitled Group";

            if (this.$root.publicGroupList.length === 0) {
                groupName = "Services";
            }

            this.$root.publicGroupList.push({
                name: groupName,
                monitorList: [],
            });
        },

        discard() {
            location.reload();
        },

        changeTheme(name) {
            this.config.statusPageTheme = name;
        },

        /**
         * Crop Success
         */
        cropSuccess(imgDataUrl) {
            this.imgDataUrl = imgDataUrl;
        },

        showImageCropUploadMethod() {
            if (this.editMode) {
                this.showImageCropUpload = true;
            }
        },

        createIncident() {
            this.enableEditIncidentMode = true;

            if (this.incident) {
                this.previousIncident = this.incident;
            }

            this.incident = {
                title: "",
                content: "",
                style: "primary",
            };
        },

        postIncident() {
            if (this.incident.title == "" || this.incident.content == "") {
                toast.error("Please input title and content.");
                return;
            }

            this.$root.getSocket().emit("postIncident", this.incident, (res) => {

                if (res.ok) {
                    this.enableEditIncidentMode = false;
                    this.incident = res.incident;
                } else {
                    toast.error(res.msg);
                }

            });

        },

        /**
         * Click Edit Button
         */
        editIncident() {
            this.enableEditIncidentMode = true;
            this.previousIncident = Object.assign({}, this.incident);
        },

        cancelIncident() {
            this.enableEditIncidentMode = false;

            if (this.previousIncident) {
                this.incident = this.previousIncident;
                this.previousIncident = null;
            }
        },

        unpinIncident() {
            this.$root.getSocket().emit("unpinIncident", () => {
                this.incident = null;
            });
        }
    }
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.overall-status {
    font-weight: bold;
    font-size: 25px;

    .ok {
        color: $primary;
    }

    .warning {
        color: $warning;
    }

    .danger {
        color: $danger;
    }
}

h1 {
    font-size: 30px;

    img {
        vertical-align: middle;
        height: 60px;
        width: 60px;
    }
}

footer {
    text-align: center;
    font-size: 14px;
}

.description span {
    min-width: 50px;
}

.logo-wrapper {
    display: inline-block;
    position: relative;

    &:hover {
        .icon-upload {
            transform: scale(1.2);
        }
    }

    .icon-upload {
        transition: all $easing-in 0.2s;
        position: absolute;
        bottom: 6px;
        font-size: 20px;
        left: -14px;
        background-color: white;
        padding: 5px;
        border-radius: 10px;
        cursor: pointer;
        box-shadow: 0 15px 70px rgba(0, 0, 0, 0.9);
    }
}

.logo {
    transition: all $easing-in 0.2s;

    &.edit-mode {
        cursor: pointer;

        &:hover {
            transform: scale(1.2);
        }
    }
}

.incident {
    .content {
        &[contenteditable=true] {
            min-height: 60px;
        }
    }

    .date {
        font-size: 12px;
    }
}

.mobile {
    h1 {
        font-size: 22px;
    }

    .overall-status {
        font-size: 20px;
    }
}

</style>
