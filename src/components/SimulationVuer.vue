<template>
  <div class="simulation-vuer" v-loading="showUserMessage" :element-loading-text="userMessage">
    <p v-if="!hasValidSimulationUiInfo && !showUserMessage" class="default error"><span class="error">Error:</span> {{ errorMessage }}.</p>
    <div class="main" v-if="hasValidSimulationUiInfo">
      <div class="main-left" :class="{'with-buttons': !libopencorSet || uuid}">
        <p class="default name" v-if="!libopencorSet">{{ name }}</p>
        <el-divider v-if="!libopencorSet"></el-divider>
        <p class="default input-parameters">Input parameters</p>
        <div class="input scrollbar">
          <SimulationVuerInput v-for="(input, index) in simulationUiInfo.input"
            ref="simInput"
            :defaultValue="input.defaultValue"
            :key="`input-${index}`"
            :name="input.name"
            :maximumValue="input.maximumValue"
            :minimumValue="input.minimumValue"
            :possibleValues="input.possibleValues"
            :stepValue="input.stepValue"
          />
        </div>
        <div class="buttons-container">
          <div class="primary-button" v-if="!libopencorSet">
            <el-button type="primary" size="small" @click="startSimulation()">Run Simulation</el-button>
          </div>
          <div class="secondary-button" v-if="uuid">
            <el-button size="small" @click="runOnOsparc()">Run on oSPARC</el-button>
          </div>
          <div class="secondary-button" v-if="!libopencorSet">
            <el-button size="small" @click="viewDataset()">View Dataset</el-button>
          </div>
          <div class="secondary-button" v-if="libopencorSet && idType === 'pmr_path'">
            <el-button size="small" @click="viewWorkspace()">View Workspace</el-button>
          </div>
          <p class="default note" v-if="uuid">Additional parameters are available on oSPARC</p>
        </div>
      </div>
      <div class="main-right" ref="output" v-show="isSimulationValid">
        <PlotVuer v-for="(_outputPlot, index) in simulationUiInfo.output.plots"
          :key="`output-${index}`"
          :metadata="plotMetadata(index)"
          :data-source="{ data: simulationResults[index] }"
          :plotLayout="layout[index]"
          :plotType="'plotly-only'"
          :selectorUi="false"
        />
      </div>
      <div class="main-right" v-show="!isSimulationValid">
        <p class="default error"><span class="error">Error:</span> <span v-html="errorMessage"></span>.</p>
      </div>
    </div>
  </div>
</template>

<script>
import { PlotVuer } from "@abi-software/plotvuer";
import "@abi-software/plotvuer/dist/style.css";
import SimulationVuerInput from "./SimulationVuerInput.vue";
import { ElButton, ElDivider, ElLoading } from "element-plus";
import { evaluateValue, finaliseUi, OPENCOR_SOLVER_NAME } from "./common.js";
import { validJson } from "./json.js";
import libOpenCOR from "https://mapcore-demo.org/current/opencor-wasm/0.0.3/libopencor.js";
import { markRaw } from "vue";
import { create, all } from "mathjs";

const LIBOPENCOR_SOLVER = "libOpenCOR";
const OSPARC_SOLVER = "oSPARC";
const PMR_URL = "https://models.physiomeproject.org/";

const math = create(all, {});

const IdType = Object.freeze({
  DATASET_ID: 'dataset_id',
  DATASET_URL: 'dataset_url',
  PMR_PATH: 'pmr_path',
  RAW_COMBINE_ARCHIVE: 'raw_combine_archive',
});

/**
 * SimulationVuer
 */
export default {
  LIBOPENCOR_SOLVER: LIBOPENCOR_SOLVER,
  OSPARC_SOLVER: OSPARC_SOLVER,
  name: "SimulationVuer",
  components: {
    PlotVuer,
    SimulationVuerInput,
    ElButton,
    ElDivider,
    ElLoading,
  },
  props: {
    /**
     * The URL of a running copy of the [SPARC API](https://github.com/nih-sparc/sparc-api).
     */
    apiLocation: {
      required: true,
      type: String,
    },
    /**
     * Either:<br/>
     * <li>the id of a SPARC simulation-based dataset (as a `Number`, e.g., `135`);</li>
     * <li>the [PMR](https://models.physiomeproject.org/) path to an [OMEX](https://combinearchive.org/) file (as a `String`, e.g., `workspace/b7c/rawfile/e0ae8d2d56aaaa091e23e1ee7e84cacbda1dfb6b/135.omex`);</li>
     * <li>the direct URL to an OMEX file (as a `String`, e.g., `https://raw.githubusercontent.com/opencor/webapp/refs/heads/main/tests/models/ui/135.omex`); or</li>
     * <li>a raw OMEX file (as a `Uint8Array`).</li>
     */
    id: {
      required: true,
      type: [Number, String, Uint8Array],
    },
  },
  data: function () {
    // Determine the ID's type.

    let idType;

    if (typeof this.id === "number") {
      idType = IdType.DATASET_ID;
    } else if (this.id instanceof Uint8Array) {
      idType = IdType.RAW_COMBINE_ARCHIVE;
    } else if (this.id.startsWith("https://")) {
      idType = IdType.DATASET_URL;
    } else {
      idType = IdType.PMR_PATH;
    }

    // Retrieve some information about the dataset.

    if (idType === IdType.DATASET_ID) {
      const xmlhttp = new XMLHttpRequest();

      xmlhttp.open("GET", this.apiLocation + "/sim/dataset/" + this.id);
      xmlhttp.onreadystatechange = () => {
        if (xmlhttp.readyState === 4) {
          if (xmlhttp.status === 200) {
            const datasetInfo = JSON.parse(xmlhttp.responseText);

            this.name = datasetInfo.name;
            this.uuid = (datasetInfo.study !== undefined) ? datasetInfo.study.uuid : undefined;
          }
        }
      };
      xmlhttp.send();
    }

    return {
      errorMessage: "",
      fileManager: undefined,
      hasFinalisedUi: false,
      hasValidSimulationUiInfo: false,
      idType: idType,
      instance: undefined,
      isMounted: false,
      isSimulationValid: true,
      layout: [],
      libopencor: undefined,
      libopencorSet: false,
      model: undefined,
      name: null,
      opencorBasedSimulation: true,
      output: undefined,
      perfectScollbarOptions: {
        suppressScrollX: true,
      },
      showUserMessage: false,
      simulationResults: {},
      simulationResultsId: {},
      simulationUiInfo: {},
      solver: undefined,
      userMessage: "",
      ui: null,
      uuid: null,
    };
  },
  methods: {
    /**
     * @public
     * Generate the metadata associated with the plot which `index` is given.
     * @arg `index`
     */
    plotMetadata(index) {
      return {
        version: "1.1.0",
        type: "plot",
        attrs: {
          style: "timeseries",
          layout: this.layout[index],
        },
      };
    },
    /**
     * @public
     * Manage the file associated with the given `url` and `fileContents`.
     * @arg `url`
     * @arg `fileContents`
     */
    manageFile(url, fileContents) {
      let file = this.fileManager.file(url);

      if (file === null) {
        file = new this.libopencor.File(url);
      }

      const fileContentsPtr = this.libopencor._malloc(fileContents.length);
      const mem = new Uint8Array(this.libopencor.HEAPU8.buffer, fileContentsPtr, fileContents.length);

      mem.set(fileContents);

      file.setContents(fileContentsPtr, fileContents.length);

      this.libopencor._free(fileContentsPtr);

      return file;
    },
    /**
     * @public
     * Run a PMR-based COMBINE archive using libOpenCOR.
     */
    runSimulation() {
      if (this.instance === undefined) {
        // Retrieve an instance of the model.

        const document = new this.libopencor.SedDocument(this.fileManager.file(PMR_URL + this.id));

        this.model = markRaw(document.model());
        this.instance = markRaw(document.instantiate());

        document.delete();
      }

      // Run the simulation after passing some changes, if any, to the model.

      this.model.removeAllChanges();

      for (const [parameter, value] of Object.entries(this.parametersData())) {
        const parameterParts = parameter.split("/")

        this.model.addChange(new this.libopencor.SedChangeAttribute(parameterParts[0], parameterParts[1], value.toString()));
      }

      this.instance.run();

      // Retrieve the simulation results.

      const res = {};
      const instanceTask = this.instance.task(0);
      let foundAllOutputs = true;
      let foundOutput;

      for (const output of this.outputData()) {
        foundOutput = false;

        if (output === instanceTask.voiName) {
          res[output] = instanceTask.voiAsArray;

          foundOutput = true;
        }

        if (res[output] === undefined) {
          for (let i = 0; i < instanceTask.stateCount; ++i) {
            if (output === instanceTask.stateName(i)) {
              res[output] = instanceTask.stateAsArray(i);

              foundOutput = true;

              break;
            }
          }
        }

        if (res[output] === undefined) {
          for (let i = 0; i < instanceTask.rateCount; ++i) {
            if (output === instanceTask.rateName(i)) {
              res[output] = instanceTask.rateAsArray(i);

              foundOutput = true;

              break;
            }
          }
        }

        if (res[output] === undefined) {
          for (let i = 0; i < instanceTask.constantCount; ++i) {
            if (output === instanceTask.constantName(i)) {
              res[output] = instanceTask.constantAsArray(i);

              foundOutput = true;

              break;
            }
          }
        }

        if (res[output] === undefined) {
          for (let i = 0; i < instanceTask.computedConstantCount; ++i) {
            if (output === instanceTask.computedConstantName(i)) {
              res[output] = instanceTask.computedConstantAsArray(i);

              foundOutput = true;

              break;
            }
          }
        }

        if (res[output] === undefined) {
          for (let i = 0; i < instanceTask.algebraicCount; ++i) {
            if (output === instanceTask.algebraicName(i)) {
              res[output] = instanceTask.algebraicAsArray(i);

              foundOutput = true;

              break;
            }
          }
        }

        if (!foundOutput) {
          console.warn("SIMULATION: output '" + output + "' could not be found.");

          foundAllOutputs = false;
        }
      }

      if (foundAllOutputs) {
        this.processSimulationResults(res);
      } else {
        this.hasValidSimulationUiInfo = false;
        this.errorMessage = "some outputs could not be found";
      }

      this.showUserMessage = false;
    },
    /**
     * @public
     * Build the simulation UI using `simulationUiInfo`, a JSON object that describes the contents of the simulation UI.
     * @arg `simulationUiInfo`
     */
    buildSimulationUi(simulationUiInfo) {
      // Keep track of the simulation UI information.

      this.simulationUiInfo = simulationUiInfo;

      // Make sure that the simulation UI information is valid.

      this.hasValidSimulationUiInfo = validJson(this.simulationUiInfo, this.libopencor === undefined);

      if (!this.hasValidSimulationUiInfo) {
        this.errorMessage = "the simulation.json file is malformed";

        return;
      }

      // Retrieve and keep track of the solver to be used for the simulation, if
      // needed.

      if (this.libopencor === undefined) {
        this.simulationUiInfo.simulation.solvers.forEach((solver) => {
          if ((solver.if === undefined) || evaluateValue(this, solver.if)) {
            this.solver = solver;
          }
        });

        if (this.solver === undefined) {
          this.hasValidSimulationUiInfo = false;
          this.errorMessage = "no solver name and/or solver version specified";

          return;
        }

        this.opencorBasedSimulation = this.solver.name === OPENCOR_SOLVER_NAME;
      }

      // Initialise our UI.

      this.simulationUiInfo.output.data.forEach((data) => {
        this.simulationResultsId[data.id] = data.name;
      });

      let index = -1;

      this.simulationUiInfo.output.plots.forEach((outputPlot) => {
        ++index;

        this.layout[index] = {
          paper_bgcolor: "rgba(0, 0, 0, 0)",
          plot_bgcolor: "rgba(0, 0, 0, 0)",
          autosize: true,
          margin: {
            t: 25,
            l: 55,
            r: 25,
            b: 30,
            pad: 4,
          },
          loading: false,
          options: {
            responsive: true,
            scrollZoom: true,
          },
          dragmode: "pan",
          xaxis: {
            title: {
              text: outputPlot.xAxisTitle,
              font: {
                size: 10,
              },
            },
          },
          yaxis: {
            title: {
              text: outputPlot.yAxisTitle,
              font: {
                size: 10,
              },
            },
          },
        };
      });

      // Finalise our UI.
      // Note: we try both here and in the mounted() function since we have no
      //       idea how long it's going to take to retrieve the simulation UI
      //       information.

      this.$nextTick(() => {
        finaliseUi(this);
      });
    },
    /**
     * @public
     * Extract the simulation UI JSON file from the given file contents and build the simulation UI.
     * @arg `libopencor`
     * @arg `fileContents`
     */
    extractAndBuildSimulationUi(libopencor, fileContents) {
      this.libopencor = markRaw(libopencor);
      this.libopencorSet = true;
      this.fileManager = markRaw(this.libopencor.FileManager.instance());

      const file = this.manageFile(PMR_URL + this.id, fileContents);

      if (file.type.value !== libopencor.File.Type.COMBINE_ARCHIVE.value) {
        this.showUserMessage = false;
      } else {
        const decoder = new TextDecoder();
        const simulationJson = file.childFileFromFileName("simulation.json");

        if (simulationJson === null) {
          this.errorMessage = "no simulation JSON file could be found";

          this.showUserMessage = false;
        } else {
          const simulationUiInfo = JSON.parse(decoder.decode(simulationJson.contents()));

          this.showUserMessage = false;

          this.$nextTick(() => {
            this.buildSimulationUi(simulationUiInfo);
          });
        }
      }
    },
    /**
     * @public
     * Run the simulation-based dataset directly on oSPARC. Not all simulation-based datasets can be run directly on
     * oSPARC, but for those that can the simulation UI shows a `Run on oSPARC` button which, when clicked, calls this
     * method.
     */
    runOnOsparc() {
      window.open(`https://osparc.io/study/${this.uuid}`, "_blank");
    },
    /**
     * @public
     * View the simulation-based dataset on the SPARC portal. The simulation UI has a `View Dataset` button which, when
     * clicked, calls this method.
     */
    viewDataset() {
      window.open(`https://sparc.science/datasets/${this.id}?type=dataset`, "_blank");
    },
    /**
     * @public
     * View the simulation-based dataset on PMR. The simulation UI has a `View Workspace` button which, when clicked,
     * calls this method.
     */
    viewWorkspace() {
      const url = PMR_URL + this.id;

      window.open(url.substring(0, url.lastIndexOf("/")), "_blank");
    },
    /**
     * @public
     * Data needed to set a model's parameters.
     */
    parametersData() {
      const res = {};

      this.simulationUiInfo.parameters.forEach((parameter) => {
        res[parameter.name] = evaluateValue(this, parameter.value);
      });

      return res;
    },
    /**
     * @public
     * Data needed to specify the model output.
     */
    outputData() {
      if (this.output === undefined) {
        if (this.simulationUiInfo.output.data !== undefined) {
          this.output = [];

          this.simulationUiInfo.output.data.forEach((output) => {
            this.output.push(output.name);
          });
        }
      }

      return this.output;
    },
    /**
     * @public
     * Create the `request` that is going to be used by `startSimulation` to ask oSPARC to start the simulation.
     */
    retrieveRequest() {
      const request = {
        solver: this.solver
      };

      if (this.opencorBasedSimulation) {
        request.opencor = {
          model_url: this.simulationUiInfo.simulation.opencor.resource,
          json_config: {},
        };

        if ((this.simulationUiInfo.simulation.opencor.endingPoint !== undefined)
          && (this.simulationUiInfo.simulation.opencor.pointInterval !== undefined)) {
          request.opencor.json_config.simulation = {
            "Ending point": this.simulationUiInfo.simulation.opencor.endingPoint,
            "Point interval": this.simulationUiInfo.simulation.opencor.pointInterval,
          };
        }

        request.opencor.json_config.parameters = this.parametersData();

        const output = this.outputData();

        if (output !== undefined) {
          request.opencor.json_config.output = output;
        }
      } else {
        request.osparc = {};

        request.osparc.job_inputs = this.parametersData();
      }

      return request;
    },
    /**
     * @public
     * Process the simulation results retrieved by `checkSimulation`. The simulation results are post-processed, if
     * needed, and then readied for use by `PlotVuer`.
     * @arg `results`
     */
    processSimulationResults(results) {
      // Convert, if needed, the results to a JSON format that is compatible
      // with our OpenCOR results.

      if (typeof (results) === "string") {
        const SPACES = /[ \t]+/g;
        const lines = results.trim().split("\n");
        const iMax = lines[0].trim().split(SPACES).length;

        results = {};

        for (let i = 0; i < iMax; ++i) {
          results[i] = [];
        }

        let i = -1;

        lines.forEach((line) => {
          ++i;

          let j = -1;
          const values = line.trim().split(SPACES);

          values.forEach((value) => {
            results[++j][i] = Number(value);
          });
        });
      }

      // Get the results ready for plotting.

      const parser = new math.parser();

      Object.keys(this.simulationResultsId).forEach((id) => {
        parser.set(id, results[this.simulationResultsId[id]]);
      });

      let index = -1;

      this.simulationUiInfo.output.plots.forEach((outputPlot) => {
        this.simulationResults[++index] = [
          {
            x: parser.evaluate(outputPlot.xValue),
            y: parser.evaluate(outputPlot.yValue),
            type: "scatter",
          },
        ];
      });
    },
    /**
     * @public
     * Show an HTTP issue using the given `xmlhttp`.
     * @arg `xmlhttp`
     */
    showHttpIssue(xmlhttp) {
      this.isSimulationValid = false;
      this.showUserMessage = false;
      this.errorMessage = xmlhttp.statusText.toLowerCase() + " (<a href='https://httpstatuses.com/" + xmlhttp.status + "/' target='_blank'>" + xmlhttp.status + "</a>)";
    },
    /**
     * @public
     * Check the progress of the simulation using the given `data`, a JSON object that contains the simulation job ID,
     * as well as the solver name and version. This method is first called by `startSimulation` and then every second by
     * itself until the simulation is finished.
     * @arg `data`
     */
    checkSimulation(data) {
      // Check the simulation.

      const xmlhttp = new XMLHttpRequest();

      xmlhttp.open("POST", this.apiLocation + "/check_simulation");
      xmlhttp.setRequestHeader("Content-type", "application/json");
      xmlhttp.onreadystatechange = () => {
        if (xmlhttp.readyState === 4) {
          if (xmlhttp.status === 200) {
            let response = JSON.parse(xmlhttp.responseText);

            this.isSimulationValid = response.status === "ok";

            if (this.isSimulationValid) {
              if (response.results !== undefined) {
                // The simulation is finished, so process its results.

                this.showUserMessage = false;

                this.processSimulationResults(response.results);
              } else {
                // The simulation is not yet finished, so check again in a
                // second.

                let that = this;

                setTimeout(function () {
                  that.checkSimulation(data);
                }, 1000);
              }
            } else {
              this.showUserMessage = false;
              this.errorMessage = response.description;
            }
          } else {
            this.showHttpIssue(xmlhttp);
          }
        }
      };
      xmlhttp.send(JSON.stringify(data));
    },
    /**
     * @public
     * Start the simulation associated with the simulation-based dataset. The simulation UI has a `Run Simulation`
     * button which, when clicked, calls this method.
     */
    startSimulation() {
      // Start the simulation (after resetting our previous simulation data, in
      // case there were sonme).

      this.userMessage = "Loading simulation results...";
      this.showUserMessage = true;

      this.$nextTick(() => {
        this.simulationResults = {};

        const xmlhttp = new XMLHttpRequest();

        xmlhttp.open("POST", this.apiLocation + "/start_simulation");
        xmlhttp.setRequestHeader("Content-type", "application/json");
        xmlhttp.onreadystatechange = () => {
          if (xmlhttp.readyState === 4) {
            if (xmlhttp.status === 200) {
              let response = JSON.parse(xmlhttp.responseText);

              this.isSimulationValid = response.status === "ok";

              if (this.isSimulationValid) {
                this.checkSimulation(response.data);
              } else {
                this.showUserMessage = false;
                this.errorMessage = response.description;
              }
            } else {
              this.showHttpIssue(xmlhttp);
            }
          }
        };
        xmlhttp.send(JSON.stringify(this.retrieveRequest()));
      });
    },
  },
  created: function () {
    // Try to retrieve the UI information.

    if (this.idType === IdType.DATASET_ID) {
      this.userMessage = "Retrieving UI information...";
      this.showUserMessage = true;

      // Retrieve and build the simulation UI.

      this.$nextTick(() => {
        const xmlhttp = new XMLHttpRequest();

        xmlhttp.open("GET", this.apiLocation + "/simulation_ui_file/" + this.id);
        xmlhttp.onreadystatechange = () => {
          if (xmlhttp.readyState === 4) {
            this.showUserMessage = false;

            if (xmlhttp.status === 200) {
              this.$nextTick(() => {
                this.buildSimulationUi(JSON.parse(xmlhttp.responseText));
              });
            } else {
              this.errorMessage = "the simulation dataset could not be retrieved";
            }
          }
        };
        xmlhttp.send();
      });
    } else if (this.idType === IdType.DATASET_URL) {
      this.userMessage = "Retrieving COMBINE archive...";
      this.showUserMessage = true;

      // Retrieve the COMBINE archive, extract the simulation UI JSON file from
      // it and then build the simulation UI.

      this.$nextTick(() => {
        const xmlhttp = new XMLHttpRequest();

        xmlhttp.open("GET", this.id);
        xmlhttp.responseType = "arraybuffer";
        xmlhttp.onreadystatechange = () => {
          if (xmlhttp.readyState === 4) {
            if (xmlhttp.status === 200) {
              libOpenCOR().then((libopencor) => {
                this.extractAndBuildSimulationUi(libopencor, new Uint8Array(xmlhttp.response));
              });
            } else {
              this.errorMessage = "the COMBINE archive could not be retrieved";
              this.showUserMessage = false;
            }
          }
        };
        xmlhttp.send();
      });
    } else if (this.idType === IdType.PMR_PATH) {
      this.userMessage = "Retrieving COMBINE archive from PMR...";
      this.showUserMessage = true;

      // Retrieve the COMBINE archive, extract the simulation UI JSON file from
      // it and then build the simulation UI.

      this.$nextTick(() => {
        const xmlhttp = new XMLHttpRequest();

        xmlhttp.open("POST", this.apiLocation + "/pmr_file");
        xmlhttp.setRequestHeader("Content-type", "application/json");
        xmlhttp.onreadystatechange = () => {
          if (xmlhttp.readyState === 4) {
            if (xmlhttp.status === 200) {
              libOpenCOR().then((libopencor) => {
                this.extractAndBuildSimulationUi(libopencor, Uint8Array.from(atob(xmlhttp.response), (c) => c.charCodeAt(0)));
              });
            } else {
              this.errorMessage = "the COMBINE archive chould not be retrieved from PMR";
              this.showUserMessage = false;
            }
          }
        };
        xmlhttp.send(JSON.stringify({ path: this.id }));
      });
    } else {
      // Extract the simulation UI JSON file from the COMBINE archive and build
      // the simulation UI.

      this.userMessage = "Retrieving COMBINE archive...";
      this.showUserMessage = true;

      this.$nextTick(() => {
        libOpenCOR().then((libopencor) => {
          this.extractAndBuildSimulationUi(libopencor, this.id);
        });
      });
    }
  },
  mounted: function () {
    // Finalise our UI.
    // Note: we try both here and in the created() function since we have no
    //       idea how long it's going to take to retrieve the simulation UI
    //       information.

    this.isMounted = true;

    finaliseUi(this);
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
.simulation-vuer {
  --el-color-primary: #8300BF;
  --el-color-primary-light-7: #DAB3EC;
  --el-color-primary-light-8: #E6CCF2;
  --el-color-primary-light-9: #F3E6F9;
}

:deep(.el-button:hover) {
  box-shadow: -3px 2px 4px #00000040;
}

:deep(.el-divider) {
  margin: -8px 0 8px 0 !important;
  width: 210px;
}

:deep(.el-loading-spinner) {
  .path {
    stroke: #8300BF;
  }

  i,
  .el-loading-text {
    color: #8300BF;
  }
}

div.input {
  display: flex;
  flex-direction: column;
  gap: 12px;
  border: 1px solid #dcdfe6;
  padding: 12px;
  min-height: 0;
  max-height: none;
  flex: 0 1 auto;
  align-self: flex-start;
  width: 100%;
  box-sizing: border-box;
}

.buttons-container {
  flex-shrink: 0;
}

div.main {
  display: grid;
  --mainLeftWidth: 243px;
  grid-template-columns: var(--mainLeftWidth) calc(100% - var(--mainLeftWidth));
  height: 100%;
  container-type: size;
}

div.main-left {
  border-right: 1px solid #dcdfe6;
  padding: 16px;
  height: 100%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  box-sizing: border-box;
}

div.main-right {
  height: 100%;
  overflow: auto;
}

div.main-right.x1 > .plotvuer_parent {
  height: 100%;
}

div.main-right.x2 > .plotvuer_parent {
  height: 50%;
}

div.main-right.x3 > .plotvuer_parent {
  height: 33.333%;
}

div.main-right.x4 > .plotvuer_parent {
  height: 25%;
}

div.main-right.x5 > .plotvuer_parent {
  height: 20%;
}

div.main-right.x6 > .plotvuer_parent {
  height: 16.667%;
}

div.main-right.x7 > .plotvuer_parent {
  height: 14.286%;
}

div.main-right.x8 > .plotvuer_parent {
  height: 12.5%;
}

div.main-right.x9 > .plotvuer_parent {
  height: 11.111%;
}

:deep(div.main-right div.controls) {
  height: 0;
}

div.primary-button,
div.secondary-button {
  display: flex;
  justify-content: flex-end;
  width: 210px;
}

div.primary-button {
  margin-top: 14px;
}

div.secondary-button {
  margin-top: 8px;
}

div.primary-button .el-button,
div.secondary-button .el-button,
div.primary-button .el-button:hover,
div.secondary-button .el-button:hover {
  width: 121px;
  border-color: #8300bf;
}

div.primary-button .el-button,
div.primary-button .el-button:hover {
  background-color: #8300bf;
}

div.secondary-button .el-button,
div.secondary-button .el-button:hover {
  background-color: #f9f2fc;
  color: #8300bf;
}

div.scrollbar {
  overflow-y: auto;
  overflow-x: hidden;
  scrollbar-width: thin;
}

div.scrollbar::-webkit-scrollbar {
  width: 8px;
  right: -8px;
  background-color: #f5f5f5;
}

div.scrollbar::-webkit-scrollbar-thumb {
  border-radius: 4px;
  box-shadow: 0 2px 4px 0 rgba(0, 0, 0, 0.06);
  background-color: #979797;
}

div.scrollbar::-webkit-scrollbar-track {
  border-radius: 10px;
  background-color: #f5f5f5;
}

div.simulation-vuer {
  height: 100%;
}

p.default {
  font-family: Asap, sans-serif;
  letter-spacing: 0;
  margin: 16px 0;
  text-align: start;
}

p.error {
  margin-left: 16px;
}

p.input-parameters {
  margin-bottom: 16px;
}

p.name,
p.input-parameters {
  margin-top: 0;
  font-weight: bold;
}

p.name {
  line-height: 20px;
}

p.note {
  font-size: 12px;
  line-height: 16px;
  margin-bottom: 0;
}

span.error {
  font-weight: bold;
}

@container (height < 400px) {
  div.main-left.with-buttons {
    @extend .scrollbar;

    p {
      font-size: 12px;
      margin: 8px 0;

      &.name,
      &.input-parameters {
        font-size: 14px;
      }
    }

    div.input {
      min-height: 180px;
    }
  }
}
</style>
