<template>
  <div
    class="simulation-vuer"
    v-loading="showUserMessage"
    :element-loading-text="userMessage"
  >
    <div class="container" v-if="opencorOmexFile === null">
      <p
        v-if="!hasValidSimulationUiInfo && !showUserMessage"
        class="default error"
      >
        <span class="error">Error:</span> {{ errorMessage }}.
      </p>
      <div class="main" v-if="hasValidSimulationUiInfo">
        <div class="main-left" :class="{ 'with-buttons': uuid }">
          <p class="default name">{{ name }}</p>
          <el-divider></el-divider>
          <p class="default input-parameters">Input parameters</p>
          <div class="input scrollbar">
            <SimulationVuerInput
              v-for="(input, index) in simulationUiInfo.input"
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
            <div class="primary-button">
              <el-button type="primary" size="small" @click="startSimulation()"
                >Run Simulation</el-button
              >
            </div>
            <div class="secondary-button" v-if="uuid">
              <el-button size="small" @click="runOnOsparc()"
                >Run on oSPARC</el-button
              >
            </div>
            <div class="secondary-button">
              <el-button size="small" @click="viewDataset()"
                >View Dataset</el-button
              >
            </div>
            <p class="default note" v-if="uuid">
              Additional parameters are available on oSPARC
            </p>
          </div>
        </div>
        <div class="main-right" ref="output" v-show="isSimulationValid">
          <PlotVuer
            v-for="(_outputPlot, index) in simulationUiInfo.output.plots"
            :key="`output-${index}`"
            :metadata="plotMetadata(index)"
            :data-source="{ data: simulationResults[index] }"
            :plotLayout="layout[index]"
            :plotType="'plotly-only'"
            :selectorUi="false"
          />
        </div>
        <div class="main-right" v-show="!isSimulationValid">
          <p class="default error">
            <span class="error">Error:</span>
            {{ errorMessage }}
            <span v-if="errorStatus">
              (<a
                :href="`https://httpstatuses.com/${errorStatus}`"
                target="_blank"
                rel="noopener noreferrer"
              >
                {{ errorStatus }} </a
              >) </span
            >.
          </p>
        </div>
      </div>
    </div>
    <div v-else class="opencor">
      <OpenCOR
        ref="opencorRef"
        :omex="opencorOmexFile"
        theme="light"
        @externalData="onExternalData($event)"
        @file="onFile($event)"
        @simulationData="onSimulationData($event)"
      />
    </div>
  </div>
</template>

<script>
import { PlotVuer } from "@abi-software/plotvuer";
import "@abi-software/plotvuer/dist/style.css";
import OpenCOR from "@opencor/opencor";
import "@opencor/opencor/style.css";

import { ElButton, ElDivider } from "element-plus";
import { create, all } from "mathjs";

import { evaluateValue, finaliseUi, OPENCOR_SOLVER_NAME } from "./common.js";
import { validJson } from "./json.js";
import SimulationVuerInput from "./SimulationVuerInput.vue";

const PMR_URL = "https://models.physiomeproject.org/";

const math = create(all, {});

const IdType = Object.freeze({
  DATASET_ID: "dataset_id",
  DATASET_URL: "dataset_url",
  PMR_PATH: "pmr_path",
  RAW_COMBINE_ARCHIVE: "raw_combine_archive",
});

function isWebProtocol(urlString) {
  try {
    const url = new URL(urlString);
    // Protocol property includes the colon, e.g., "https:".
    return url.protocol === "http:" || url.protocol === "https:";
  } catch (_err) {
    // String was not a valid URL.
    return false;
  }
}

/**
 * SimulationVuer
 */
export default {
  name: "SimulationVuer",
  components: {
    PlotVuer,
    SimulationVuerInput,
    ElButton,
    ElDivider,
    OpenCOR,
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
    } else if (isWebProtocol(this.id)) {
      idType = IdType.DATASET_URL;
    } else {
      idType = IdType.PMR_PATH;
    }

    // Retrieve some information about the dataset.

    if (idType === IdType.DATASET_ID) {
      const xmlhttp = new XMLHttpRequest();

      xmlhttp.open("GET", `${this.apiLocation}/sim/dataset/${this.id}`);
      xmlhttp.onreadystatechange = () => {
        if (xmlhttp.readyState === 4) {
          if (xmlhttp.status === 200) {
            const datasetInfo = JSON.parse(xmlhttp.responseText);

            this.name = datasetInfo.name;
            this.uuid =
              datasetInfo.study !== undefined
                ? datasetInfo.study.uuid
                : undefined;
          }
        }
      };
      xmlhttp.send();
    }

    return {
      errorMessage: "",
      errorStatus: null,
      fileManager: undefined,
      hasFinalisedUi: false,
      hasValidSimulationUiInfo: false,
      idType: idType,
      instance: undefined,
      isMounted: false,
      isSimulationValid: true,
      layout: [],
      model: undefined,
      name: null,
      opencorBasedSimulation: true,
      opencorOmexFile: null,
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
      activeSubscriptions: [],
    };
  },
  methods: {
    /**
     * @public
     * Add a data subscription.
     * @param `subscription`
     */
    addDataSubscription(subscription) {
      // Check that the subscription is valid.

      if (!subscription || typeof subscription !== "object") {
        console.warn(
          "SimulationVuer: addDataSubscription: subscription must be an object.",
        );

        return;
      }

      // Check that the subscription has the expected ID.

      const EXPECTED_ID = "nz.ac.auckland.simulation-data-request";
      const EXPECTED_MAJOR_VERSION = 0;

      if (subscription.id !== EXPECTED_ID) {
        console.warn(
          `SimulationVuer: addDataSubscription: invalid ID (expected '${EXPECTED_ID}' but got '${subscription.id}').`,
        );

        return;
      }

      // Check that the version is valid and compatible with what we expect.

      if (typeof subscription.version !== "string") {
        console.warn(
          `SimulationVuer: addDataSubscription: missing or non-string version ('${subscription.version}').`,
        );

        return;
      }

      const versionParts = subscription.version.split(".");

      if (versionParts.length < 1 || !/^[0-9]+$/.test(versionParts[0])) {
        console.warn(
          `SimulationVuer: addDataSubscription: malformed version ('${subscription.version}').`,
        );

        return;
      }

      const subscriptionMajorVersion = parseInt(versionParts[0], 10);

      if (Number.isNaN(subscriptionMajorVersion)) {
        console.warn(
          `SimulationVuer: addDataSubscription: could not parse the major version from ('${subscription.version}')`,
        );

        return;
      }

      if (subscriptionMajorVersion !== EXPECTED_MAJOR_VERSION) {
        console.warn(
          `SimulationVuer: addDataSubscription: version mismatch (expected v${EXPECTED_MAJOR_VERSION}.y.z but got v${subscription.version}).`,
        );

        return;
      }

      // Check that the payload contains the required fields.

      const payload = subscription.payload || {};
      const missing = [];

      if (payload.windowId == null) {
        missing.push("windowId");
      }

      if (payload.ownerId == null) {
        missing.push("ownerId");
      }

      if (!payload.component) {
        missing.push("component");
      }

      if (!payload.variable) {
        missing.push("variable");
      }

      if (missing.length) {
        console.warn(
          `SimulationVuer: addDataSubscription: payload missing fields: ${missing.join(", ")}.`,
        );

        return;
      }

      // The subscription is valid, so add it to our list of active subscriptions.

      this.activeSubscriptions.push(payload);

      // Ask OpenCOR to track the simulation data associated with the subscription's component and variable (and the
      // VOI, if requested).

      const modelParameters = [];

      if (subscription.payload?.withVOI) {
        modelParameters.push("VOI");
      }

      modelParameters.push(
        `${subscription.payload?.component}/${subscription.payload?.variable}`,
      );

      this.$refs.opencorRef?.trackSimulationData(modelParameters);
    },
    /**
     * @public
     * Add external data to the OpenCOR simulation.
     * @param `csv` A CSV string or URL for the external data.
     * @param `voiExpression` Optional VOI expression to map the data.
     * @param `modelParameters` An array of model parameter identifiers.
     */
    addExternalData(csv, voiExpression, modelParameters) {
      if (!this.$refs.opencorRef?.addExternalData) {
        return Promise.reject(
          new Error("SimulationVuer: OpenCOR instance is not available."),
        );
      }

      return this.$refs.opencorRef.addExternalData(csv, voiExpression, modelParameters);
    },
    /**
     * @public
     * Remove a data subscription.
     * @param `subscriptionId`
     */
    removeDataSubscription(subscriptionId) {
      // Ask OpenCOR to stop tracking the simulation data associated with the subscription's component and variable (and
      // the VOI, if requested and unless it's requested by another subscription).

      const subscription = this.activeSubscriptions.find(
        (activeSubscription) => {
          return activeSubscription.windowId === subscriptionId;
        },
      );

      if (!subscription) {
        console.warn(
          `SimulationVuer: removeDataSubscription: no active subscription found for id ${subscriptionId}.`,
        );

        return;
      }

      const isVoiTrackedByAnotherSubscription = this.activeSubscriptions.some(
        (activeSubscription) => {
          return (
            activeSubscription.windowId !== subscriptionId &&
            activeSubscription.withVOI
          );
        },
      );
      const isModelParameterTrackedByAnotherSubscription =
        this.activeSubscriptions.some((activeSubscription) => {
          return (
            activeSubscription.windowId !== subscriptionId &&
            activeSubscription.component === subscription.component &&
            activeSubscription.variable === subscription.variable
          );
        });

      if (
        !isVoiTrackedByAnotherSubscription ||
        !isModelParameterTrackedByAnotherSubscription
      ) {
        const modelParameters = [];

        if (subscription.withVOI && !isVoiTrackedByAnotherSubscription) {
          modelParameters.push("VOI");
        }

        if (!isModelParameterTrackedByAnotherSubscription) {
          modelParameters.push(
            `${subscription.component}/${subscription.variable}`,
          );
        }

        if (modelParameters.length) {
          this.$refs.opencorRef?.untrackSimulationData(modelParameters);
        }
      }

      // Remove the subscription from our list of active subscriptions.

      this.activeSubscriptions = this.activeSubscriptions.filter(
        (activeSubscription) => {
          return activeSubscription.windowId !== subscriptionId;
        },
      );
    },
    /**
     * @public
     * Let the outside world know that we have received some external data from OpenCOR by emitting an `externalData` event.
     * @param `event`
     */
    onExternalData(event) {
      this.$emit("externalData", event);
    },
    /**
     * @public
     * Let the outside world know that we have received a file from OpenCOR by emitting a `file` event.
     * @param `event`
     */
    onFile(event) {
      this.$emit("file", event);
    },
    /**
     * @public
     * Let the outside world know that we have received some simulation data from OpenCOR by emitting a `simulationData`
     * event, as well as a `data-notification` event.
     * @param `event`
     */
    onSimulationData(event) {
      const simulationData = event.simulationData || {};
      let voi;

      this.activeSubscriptions.forEach((activeSubscription) => {
        const modelParameter = `${activeSubscription.component}/${activeSubscription.variable}`;
        const simData = simulationData[modelParameter];

        if (simData == null) {
          console.warn(
            `SimulationVuer: onSimulationData: no data for ${modelParameter}.`,
          );

          return;
        }

        const data = {
          y: simData.data,
          title: `${activeSubscription.component}.${activeSubscription.variable} (${simData.unit})`,
        };

        if (activeSubscription.withVOI) {
          if (simulationData.VOI == null) {
            console.warn("SimulationVuer: onSimulationData: no data for VOI.");

            return;
          } else {
            voi ??= simulationData.VOI;

            data.x = voi;
          }
        }

        this.$emit("data-notification", {
          id: "nz.ac.auckland.simulation-data-response",
          version: "0.1.0",
          payload: {
            windowId: activeSubscription.windowId,
            ownerId: activeSubscription.ownerId,
            data,
          },
        });
      });

      this.$emit("simulationData", event);
    },
    /**
     * @public
     * Generate the metadata associated with the plot which `index` is given.
     * @param `index`
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
     * Build the simulation UI using `simulationUiInfo`, a JSON object that describes the contents of the simulation UI.
     * @param `simulationUiInfo`
     */
    buildSimulationUi(simulationUiInfo) {
      // Keep track of the simulation UI information.

      this.simulationUiInfo = simulationUiInfo;

      // Make sure that the simulation UI information is valid.

      this.hasValidSimulationUiInfo = validJson(this.simulationUiInfo);

      if (!this.hasValidSimulationUiInfo) {
        this.errorMessage = "the simulation.json file is malformed";

        return;
      }

      // Retrieve and keep track of the solver to be used for the simulation.

      this.simulationUiInfo.simulation.solvers.forEach((solver) => {
        if (solver.if === undefined || evaluateValue(this, solver.if)) {
          this.solver = solver;
        }
      });

      if (this.solver === undefined) {
        this.hasValidSimulationUiInfo = false;
        this.errorMessage = "no solver name and/or solver version specified";

        return;
      }

      this.opencorBasedSimulation = this.solver.name === OPENCOR_SOLVER_NAME;

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
     * @private
     * Open the given `url` in a new browser tab, making sure to do so safely.
     * @param `url`
     */
    openUrl(url) {
      const a = document.createElement("a");

      a.href = url;
      a.target = "_blank";
      a.rel = "noopener noreferrer";
      a.style.display = "none";

      document.body.appendChild(a);

      a.click();

      document.body.removeChild(a);
    },
    /**
     * @public
     * Run the simulation-based dataset directly on oSPARC. Not all simulation-based datasets can be run directly on
     * oSPARC, but for those that can the simulation UI shows a `Run on oSPARC` button which, when clicked, calls this
     * method.
     */
    runOnOsparc() {
      this.openUrl(`https://osparc.io/study/${this.uuid}`);
    },
    /**
     * @public
     * View the simulation-based dataset on the SPARC portal. The simulation UI has a `View Dataset` button which, when
     * clicked, calls this method.
     */
    viewDataset() {
      this.openUrl(`https://sparc.science/datasets/${this.id}?type=dataset`);
    },
    /**
     * @public
     * View the simulation-based dataset on PMR. The simulation UI has a `View Workspace` button which, when clicked,
     * calls this method.
     */
    viewWorkspace() {
      const url = PMR_URL + this.id;
      this.openUrl(url.substring(0, url.lastIndexOf("/")));
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
        solver: this.solver,
      };

      if (this.opencorBasedSimulation) {
        request.opencor = {
          model_url: this.simulationUiInfo.simulation.opencor.resource,
          json_config: {},
        };

        if (
          this.simulationUiInfo.simulation.opencor.endingPoint !== undefined &&
          this.simulationUiInfo.simulation.opencor.pointInterval !== undefined
        ) {
          request.opencor.json_config.simulation = {
            "Ending point":
              this.simulationUiInfo.simulation.opencor.endingPoint,
            "Point interval":
              this.simulationUiInfo.simulation.opencor.pointInterval,
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
     * @param `results`
     */
    processSimulationResults(results) {
      // Convert, if needed, the results to a JSON format that is compatible
      // with our OpenCOR results.

      if (typeof results === "string") {
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
     * @param `xmlhttp`
     */
    showHttpIssue(xmlhttp) {
      this.isSimulationValid = false;
      this.showUserMessage = false;
      this.errorMessage = xmlhttp.statusText.toLowerCase();
      this.errorStatus = xmlhttp.status;
    },
    /**
     * @public
     * Check the progress of the simulation using the given `data`, a JSON object that contains the simulation job ID,
     * as well as the solver name and version. This method is first called by `startSimulation` and then every second by
     * itself until the simulation is finished.
     * @param `data`
     */
    checkSimulation(data) {
      // Check the simulation.

      const xmlhttp = new XMLHttpRequest();

      xmlhttp.open("POST", `${this.apiLocation}/check_simulation`);
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

                setTimeout(() => {
                  that.checkSimulation(data);
                }, 1000);
              }
            } else {
              this.showUserMessage = false;
              this.errorMessage = response.description;
              this.errorStatus = null;
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

        xmlhttp.open("POST", `${this.apiLocation}/start_simulation`);
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
                this.errorStatus = null;
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

        xmlhttp.open(
          "GET",
          `${this.apiLocation}/simulation_ui_file/${this.id}`,
        );
        xmlhttp.onreadystatechange = () => {
          if (xmlhttp.readyState === 4) {
            this.showUserMessage = false;

            if (xmlhttp.status === 200) {
              this.$nextTick(() => {
                this.buildSimulationUi(JSON.parse(xmlhttp.responseText));
              });
            } else {
              this.errorMessage =
                "the simulation dataset could not be retrieved";
              this.errorStatus = null;
            }
          }
        };
        xmlhttp.send();
      });
    } else if (this.idType === IdType.DATASET_URL) {
      this.opencorOmexFile = this.id;
    } else if (this.idType === IdType.PMR_PATH) {
      this.opencorOmexFile = `${PMR_URL}${this.id}`;
    } else {
      // IdType.RAW_COMBINE_ARCHIVE
      this.opencorOmexFile = this.id;
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
  --el-color-primary: #8300bf;
  --el-color-primary-light-7: #dab3ec;
  --el-color-primary-light-8: #e6ccf2;
  --el-color-primary-light-9: #f3e6f9;
}

.buttons-container {
  flex-shrink: 0;
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
    stroke: #8300bf;
  }

  i,
  .el-loading-text {
    color: #8300bf;
  }
}

.opencor {
  line-height: 1.5;
}

:deep(.p-floatlabel:has(input:focus)) label,
:deep(.p-floatlabel:has(input:-webkit-autofill)) label,
:deep(.p-floatlabel:has(textarea:focus)) label,
:deep(.p-floatlabel:has(.p-inputwrapper-focus)) label {
  color: #8300bf;
}

:deep(.p-inputnumber-button) {
  background-color: transparent;
  border: 0;
  padding: 0;
}

:deep(.p-inputtext:enabled:focus) {
  border-color: #8300bf;
}

:deep(.p-progressbar-value) {
  background-color: #8300bf !important;
}

:deep(.p-select:not(.p-disabled).p-focus) {
  border-color: #8300bf;
}

:deep(.p-slider-range) {
  background-color: #8300bf;
}

:deep(.pi) {
  line-height: 20px;
}

:deep(div.empty-state) > p {
  margin: 0;
  line-height: 16px;
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

div.container {
  height: 100%;
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

div.opencor {
  height: 100%;
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
  margin-bottom: 8px;
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

    div.input {
      min-height: 180px;
    }
  }
}
</style>

<style>
/* Note: not sure why, but the following rules need to be global!? */

.p-contextmenu-item-label,
.p-select-option-label {
  font-family:
    Inter,
    -apple-system,
    BlinkMacSystemFont,
    "Segoe UI",
    Roboto,
    Oxygen,
    Ubuntu,
    Cantarell,
    "Fira Sans",
    "Droid Sans",
    "Helvetica Neue",
    sans-serif;
  font-size: 0.875rem;
}

.p-select-option:not(.p-select-option-selected):not(.p-disabled).p-focus {
  background: #f5f7fa !important;
}

.p-select-option.p-select-option-selected.p-focus {
  background: #f5f7fa !important;
  color: #8300bf !important;
}

.p-select-option.p-select-option-selected {
  background: white !important;
  color: #8300bf !important;
}
</style>
