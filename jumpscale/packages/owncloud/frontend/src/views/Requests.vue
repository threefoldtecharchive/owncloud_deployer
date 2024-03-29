<template>
  <div>
    <Navbar />
    <BalanceCard @setBalance="setBalance" />
    <v-dialog transition="dialog-top-transition" v-model="dialog">
      <template>
        <v-card>
          <v-card-text>
            <div class="text-h5 pa-6 card">
              <pre><code v-html="message"></code></pre>
            </div>
          </v-card-text>
          <v-card-actions class="justify-end">
            <v-btn text @click="dialog = false">Close</v-btn>
          </v-card-actions>
        </v-card>
      </template>
    </v-dialog>

    <h4 class="h4 pa-5">Requests:</h4>
    <v-data-table
      class="ma-5 elevation-1"
      :headers="headers"
      :items="requestsWithIndex"
      item-key="tname"
      show-select
      :loading="isLoading"
      loading-text="Loading... Please wait"
      v-model="selected"
      :items-per-page="15"
      @toggle-select-all="selectAllToggle"
      @click:row="handleClick"
    >
      <template v-slot:item.index="{ item }">{{ item.index }}</template>
      <template v-slot:item.email="{ item }">{{
        item.email == "" ? "-" : item.email
      }}</template>
      <template v-slot:item.status="{ item }">
        <td>
          <v-chip class="status" :class="item.status">{{ item.status }}</v-chip>
          <v-icon
            v-if="
              item.status == 'APPLY_FAILURE' || item.status == 'DESTROY_FAILURE'
            "
            color="red"
            right
          >
            mdi-information-outline</v-icon
          >
        </td>
      </template>
      <template v-slot:item.rtime="{ item }">{{ time(item.time) }}</template>
      <template v-slot:item.dtime="{ item }">{{
        time(item.deployment_timestamp)
      }}</template>
      <template v-slot:item.etime="{ item }">{{
        time(item.expired_timestamp)
      }}</template>
      <template v-slot:item.data-table-select="{ item, isSelected, select }">
        <v-simple-checkbox
          :value="isSelected"
          :readonly="!statusChecker(item.status) || balance < 1000"
          :disabled="!statusChecker(item.status) || balance < 1000"
          @input="select($event)"
        ></v-simple-checkbox>
      </template>
    </v-data-table>
    <div class="text-center pt-2 mt-10">
      <v-btn class="mr-2 bg-blue white--text" @click="deploy(selected)"
        ><v-icon left> mdi-cloud-upload</v-icon> Deploy</v-btn
      >
      <v-btn class="bg-blue white--text" @click="exportData()"
        ><v-icon left> mdi-export-variant</v-icon>Export</v-btn
      >
    </div>
  </div>
</template>

<script>
import Navbar from "@/components/Navbar.vue";
import BalanceCard from "@/components/BalanceCard.vue";
import Service from "../services/Services";
import moment from "moment";
export default {
  components: {
    BalanceCard,
    Navbar,
  },
  data() {
    return {
      headers: [
        { text: "#", value: "index" },
        { text: "Name", value: "tname" },
        { text: "Email", value: "email" },
        { text: "Status", value: "status" },
        { text: "Request time", value: "rtime" },
        { text: "Deploy time", value: "dtime" },
        { text: "Expire time", value: "etime" },
        {
          text: "Admin Selection",
          value: "data-table-select",
          sortable: false,
        },
      ],
      isLoading: true,
      requests: [],
      selected: [],
      disabled: false,
      disabledCount: 0,
      balance: null,
      dialog: false,
      message: "",
      timer: "",
    };
  },
  methods: {
    selectAllToggle() {
      if (
        this.selected.length !=
        this.requestsWithIndex.length - this.disabledCount
      ) {
        this.selected = [];
        this.requestsWithIndex.forEach((item) => {
          if (this.statusChecker(item.status)) {
            this.selected.push(item);
          }
        });
      } else this.selected = [];
    },
    getRequests() {
      Service.getRequests()
        .then((response) => {
          this.requests = response.data;
          this.disabledItems();
          this.isLoading = false;
        })
        .catch((error) => {
          console.log("Error! Could not reach the API. " + error);
        })
    },
    deploy(list) {
      if (list.length == 0) {
        alert("No items selected");
      } else {
        this.isLoading = true;
        let selectedNames = list.map((item) => item.tname);
        Service.deploy(selectedNames)
          .then(() => {
            this.getRequests();
            this.selected = [];
            list = [];
            this.isLoading = false;
          })
          .catch((error) => {
            console.log("Error! Could not reach the API. " + error);
          });
      }
    },
    exportData() {
      Service.exportData()
        .then((response) => {
          this.exportCSVFile(response.data);
        })
        .catch((error) => {
          console.log("Error! Could not reach the API. " + error);
        });
    },

    exportCSVFile(items) {
      // Convert Object to JSON
      const date = new Date().toDateString().replaceAll(" ", "");
      const time = new Date()
        .toLocaleTimeString()
        .replaceAll(":", "_")
        .replace(" ", "");
      var exportedFilenmae = `backup_${date}_${time}.csv`;

      var blob = new Blob([items], { type: "text/csv;charset=utf-8;" });
      if (navigator.msSaveBlob) {
        // IE 10+
        navigator.msSaveBlob(blob, exportedFilenmae);
      } else {
        var link = document.createElement("a");
        if (link.download !== undefined) {
          // feature detection
          // Browsers that support HTML5 download attribute
          var url = URL.createObjectURL(blob);
          link.setAttribute("href", url);
          link.setAttribute("download", exportedFilenmae);
          link.style.visibility = "hidden";
          document.body.appendChild(link);
          link.click();
          document.body.removeChild(link);
        }
      }
    },

    convertToCSV(objArray) {
      var array = typeof objArray != "object" ? JSON.parse(objArray) : objArray;
      var str = "";

      for (var i = 0; i < array.length; i++) {
        var line = "";
        for (var index in array[i]) {
          if (line != "") line += ",";

          line += array[i][index];
        }

        str += line + "\r\n";
      }

      return str;
    },
    time(ts) {
      if (ts) {
        var now = new Date();
        var timestamp = moment.unix(now / 1000);
        return timestamp.to(ts * 1000);
      }
      return "-";
    },
    setBalance(data) {
      this.balance = data;
    },
    disabledItems() {
      this.requestsWithIndex.map((item) => {
        if (!this.statusChecker(item.status)) this.disabledCount += 1;
      });
    },
    statusChecker(status) {
      if (status == "NEW" || status == "APPLY_FAILURE") {
        return true;
      }
    },
    handleClick(value) {
      if (value.error_message) {
        this.dialog = true;
        this.message = value.error_message;
      }
    },
    cancelAutoUpdate () {
            clearInterval(this.timer);
            this.timer = "";
    }
  },
  computed: {
    requestsWithIndex() {
      return this.requests.map((requests, index) => ({
        ...requests,
        index: index + 1,
      }));
    },
  },
  created() {
    this.getRequests();
    this.timer = setInterval(this.getRequests, 10000);
  },
  destroyed() {
    this.cancelAutoUpdate();
  },
};
</script>

<style >
.v-chip {
  color: #fff !important;
}
.v-chip.v-size--default {
  border-radius: 2px;
  height: 25px;
}
.v-chip.NEW {
  background-color: #2dccff !important;
}
.v-chip.PENDING {
  background-color: #9ea7ad !important;
}
.v-chip.DEPLOYING {
  background-color: #9ea7ad !important;
}
.v-chip.DESTROYING {
  background-color: #6a6f72 !important;
}
.v-chip.DEPLOYED {
  background-color: #56f000 !important;
}
.v-chip.EXPIRED {
  background-color: #000000 !important;
}
.v-chip.APPLY_FAILURE {
  background-color: #ff3838 !important;
}
.v-chip.DESTROY_FAILURE {
  background-color: #690404 !important;
}
.bg-blue {
  background-color: #041e42 !important;
}
.v-dialog:not(.v-dialog--fullscreen) {
  max-height: 75%;
}
.v-dialog {
  max-width: 50%;
}
.v-dialog > .v-card > .v-card__text {
  padding: 0;
}
pre {
  background-color: black;
  color: white;
  font-size: medium;
  font-family: Consolas, Monaco, Lucida Console, Liberation Mono,
    DejaVu Sans Mono, Bitstream Vera Sans Mono, Courier New, monospace;
  display: inline-block;
  width: 100%;
  white-space: pre-wrap;
  word-wrap: break-word;
}
</style>