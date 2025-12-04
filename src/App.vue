<template>
  <div id="app">
    <div class="app">
      <h1>SimulationVuer</h1>
      <div v-if="datasetIds.length !== 0">
        <h3>Dataset IDs (run on o²S²PARC):</h3>
        <el-radio-group v-model="activeResource" size="small">
          <el-radio-button :class="className(datasetId.id)" v-for="datasetId in datasetIds" v-bind:key="datasetId.id" :label="datasetId.label" :value="datasetId.id" />
        </el-radio-group>
      </div>
      <div v-if="datasetUrls.length !== 0">
        <h3>Dataset URLs (run in the browser):</h3>
        <el-radio-group v-model="activeResource" size="small">
          <el-radio-button :class="className(datasetUrl.id)" v-for="datasetUrl in datasetUrls" v-bind:key="datasetUrl.id" :label="datasetUrl.label" :value="datasetUrl.id" />
        </el-radio-group>
      </div>
      <div v-if="pmrPaths.length !== 0">
        <h3>PMR paths (run in the browser):</h3>
        <el-radio-group v-model="activeResource" size="small">
          <el-radio-button :class="className(pmrPath.id)" v-for="pmrPath in pmrPaths" v-bind:key="pmrPath.id" :label="pmrPath.label" :value="pmrPath.id" />
        </el-radio-group>
      </div>
      <div ref="dropArea" class="drop-area" @dragenter.prevent="onDragEnter" @dragover.prevent @drop.prevent="onDrop" @dragleave.prevent="onDragLeave">
        You can drag and drop a COMBINE archive here.
      </div>
      <el-dialog v-model="dragAndDropWarningVisible" width="352">
        <span>Please only drag and drop one COMBINE archive.</span>
        <template #footer>
          <div class="dialog-footer">
            <el-button type="primary" @click="dragAndDropWarningVisible = false">
              OK
            </el-button>
          </div>
        </template>
      </el-dialog>
    </div>
    <hr />
    <div v-for="resource in resources()" v-bind:key="resource.id">
      <div v-if="isResourceInitialised(resource.id)" v-show="resource.id === activeResource">
        <div v-if="typeof resource.id === 'number'">
          <span>
            <strong>Dataset <a :href="datasetUrl(resource.id)" target="_blank">{{ resource.id }}</a>:</strong> {{ resource.description }}
          </span>
          <hr />
        </div>
        <SimulationVuer :apiLocation="apiLocation" :id="resource.id" style="height: 640px;" />
      </div>
    </div>
    <div v-if="activeResource === -1">
      <div v-for="combineArchive in combineArchives" v-bind:key="combineArchive">
        <SimulationVuer :apiLocation="apiLocation" :id="combineArchive" style="height: 640px;" />
      </div>
    </div>
    <hr />
  </div>
</template>

<script>
import SimulationVuer from "./components/SimulationVuer.vue";
import { ElRadioButton, ElRadioGroup } from "element-plus";

export default {
  name: "App",
  components: {
    SimulationVuer,
    ElRadioButton,
    ElRadioGroup,
  },
  data: function () {
    return {
      apiLocation: import.meta.env.VITE_API_LOCATION,
      dropAreaCounter: 0,
      dragAndDropWarningVisible: false,
      combineArchives: [],
      datasetIds: [
        { id: 0, label: "Invalid", description: "Non-simulation dataset" },
        { id: 135, label: "135", description: "Computational analysis of the human sinus node action potential - Model development and effects of mutations" },
        { id: 157, label: "157", description: "Fabbri-based composite SAN model" },
        { id: 308, label: "308", description: "Kember Cardiac Nerve Model" },
        { id: 318, label: "318", description: "Multi-scale rabbit cardiac electrophysiology models" },
        { id: 320, label: "320", description: "Multi-scale human cardiac electrophysiology models" },
      ],
      datasetUrls: [
        { id: "https://raw.githubusercontent.com/opencor/webapp/refs/heads/main/tests/models/ui/invalid.omex", label: "Invalid", description: "COMBINE archive" },
        { id: "https://raw.githubusercontent.com/opencor/webapp/refs/heads/main/tests/models/ui/135.omex", label: "135", description: "COMBINE archive" },
        { id: "https://raw.githubusercontent.com/opencor/webapp/refs/heads/main/tests/models/ui/157.omex", label: "157", description: "COMBINE archive" },
        { id: "https://raw.githubusercontent.com/opencor/webapp/refs/heads/main/tests/models/ui/lorenz.omex", label: "Lorenz", description: "COMBINE archive" },
        { id: "https://raw.githubusercontent.com/opencor/webapp/refs/heads/main/tests/models/ui/tt04.omex", label: "TT04", description: "COMBINE archive" },
        { id: "https://raw.githubusercontent.com/opencor/webapp/refs/heads/main/tests/models/ui/cvs.omex", label: "CVS", description: "COMBINE archive" },
      ],
      pmrPaths: [
        { id: "workspace/b7c/rawfile/e0ae8d2d56aaaa091e23e1ee7e84cacbda1dfb6b/invalid.omex", label: "Invalid", description: "COMBINE archive from PMR" },
        { id: "workspace/b7c/rawfile/e0ae8d2d56aaaa091e23e1ee7e84cacbda1dfb6b/135.omex", label: "135", description: "COMBINE archive from PMR" },
        { id: "workspace/b7c/rawfile/e0ae8d2d56aaaa091e23e1ee7e84cacbda1dfb6b/157.omex", label: "157", description: "COMBINE archive from PMR" },
        { id: "workspace/b7c/rawfile/e0ae8d2d56aaaa091e23e1ee7e84cacbda1dfb6b/lorenz.omex", label: "Lorenz", description: "COMBINE archive from PMR" },
        { id: "workspace/b7c/rawfile/e0ae8d2d56aaaa091e23e1ee7e84cacbda1dfb6b/tt04.omex", label: "TT04", description: "COMBINE archive from PMR" },
        { id: "workspace/b7c/rawfile/e0ae8d2d56aaaa091e23e1ee7e84cacbda1dfb6b/cvs.omex", label: "CVS", description: "COMBINE archive from PMR" },
      ],
      activeResource: 0,
      initialisedResources: [],
    };
  },
  methods: {
    className(id) {
      return (   ((this.datasetIds.length !== 0) && (id === this.datasetIds[0].id))
              || ((this.datasetUrls.length !== 0) && (id === this.datasetUrls[0].id))
              || ((this.pmrPaths.length !== 0) && (id === this.pmrPaths[0].id))) ?
              "first-resource" :
              "not-first-resource";
    },
    resources() {
      return this.datasetIds.concat(this.datasetUrls).concat(this.pmrPaths);
    },
    datasetUrl(id) {
      return `https://sparc.science/datasets/${id}?type=dataset`;
    },
    isResourceInitialised(resource) {
      if (this.initialisedResources.includes(resource)) {
        return true;
      }

      if (this.activeResource === resource) {
        this.initialisedResources.push(resource);
      }

      return this.initialisedResources.includes(resource);
    },
    onDragEnter() {
      this.dropAreaCounter += 1;

      if (this.dropAreaCounter === 1) {
        this.$refs.dropArea.classList.add('drop-area-active');
      }
    },
    onDrop(event) {
      this.dropAreaCounter = 0;

      this.$refs.dropArea.classList.remove('drop-area-active');

      const files = event.dataTransfer.files;

      if (files.length !== 1) {
        this.dragAndDropWarningVisible = true;
      } else {
        files[0]
          .arrayBuffer()
          .then((arrayBuffer) => {
            this.activeResource = -1;
            this.combineArchives = [new Uint8Array(arrayBuffer)];
          })
      }
    },
    onDragLeave() {
      this.dropAreaCounter -= 1;

      if (this.dropAreaCounter === 0) {
        this.$refs.dropArea.classList.remove('drop-area-active');
      }
    }
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}

.drop-area {
  position: absolute;
  top: 8px;
  right: 8px;
  width: 192px;
  border: 2px dashed #8300bf;
  background-color: #f9f2fc;
  color: #8300bf;
  border-radius: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 8px;
}

.drop-area-active {
  background-color: #8300bf;
  color: white;
}

.el-button,
.el-button:hover {
  border-color: #8300bf;
  background-color: #8300bf;
}

.el-radio-button__inner {
  background-color: #f9f2fc;
  border-color: #8300bf;
  box-shadow: -1px 0 0 0 #8300bf !important;
  color: #8300bf;
}

.first-resource>.el-radio-button__inner {
  border-left: 1px solid #8300bf !important;
}

.not-first-resource>.el-radio-button__inner {
  border-left: 0 !important;
}

.el-radio-button__original-radio:checked+.el-radio-button__inner {
  background-color: #8300bf !important;
  border-color: #8300bf !important;
  color: white;
}

.el-radio-button__inner:hover {
  color: #8300bf;
}

a {
  color: #2c3e50;
}

div.app {
  text-align: center;
  margin-bottom: 8px;
}

h3 {
  margin-bottom: 0;
}
</style>
