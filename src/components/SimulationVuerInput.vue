<template>
  <div v-show="visible">
    <p :class="labelClasses">{{ name }}</p>
    <el-select
      v-if="isDiscrete"
      class="discrete"
      popper-class="discrete-popper"
      size="small"
      v-model="vModel"
      :teleported="true"
      @change="updateUi()"
    >
      <el-option
        v-for="possibleValue in possibleValues"
        :key="possibleValue.value"
        :label="possibleValue.name"
        :value="possibleValue.value"
      />
    </el-select>
    <div class="sliders-and-fields" v-if="!isDiscrete">
      <el-slider
        v-model="vModel"
        :max="maximumValue"
        :min="minimumValue"
        :show-input="false"
        :show-tooltip="false"
        :step="stepValue"
        @input="updateUi()"
      />
      <el-input-number
        class="scalar"
        size="small"
        v-model="vModel"
        :controls="false"
        :max="maximumValue"
        :min="minimumValue"
        :step="stepValue"
        :step-strictly="true"
        @input="updateUi()"
      />
    </div>
  </div>
</template>

<script>
import { ElInputNumber, ElOption, ElSelect, ElSlider } from "element-plus";
import { updateUi } from "./common.js";

export default {
  name: "SimulationVuerInput",
  components: {
    ElInputNumber,
    ElOption,
    ElSelect,
    ElSlider,
  },
  props: {
    defaultValue: {
      required: true,
      type: Number,
    },
    maximumValue: {
      type: Number,
    },
    minimumValue: {
      type: Number,
    },
    name: {
      required: true,
      type: String,
    },
    possibleValues: {
      type: Array,
    },
    stepValue: {
      type: Number,
    },
  },
  data: function () {
    return {
      isDiscrete: this.possibleValues !== undefined,
      labelClasses: "default " + ((this.possibleValues !== undefined) ? "discrete" : "scalar"),
      visible: true,
      vModel: this.defaultValue,
    };
  },
  methods: {
    updateUi: function () {
      updateUi(this.$parent);
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
// select box, slider and input number height
$input-height: 32px;
$input-number-width: 45px;

p {
  font-size: 14px;
}

:deep(.el-input-number.scalar) {
  width: $input-number-width;
  height: $input-height;
}

:deep(.el-input-number.scalar .el-input) {
  width: 60px;
}

:deep(.el-input-number.scalar .el-input__inner:focus) {
  border-color: #8300bf;
}

:deep(.el-select.discrete) {
  margin-top: 4px;

  .el-select__wrapper {
    height: $input-height;
  }
}

:deep(.el-slider) {
  margin-left: 8px;
  width: calc(100% - 16px);
  height: $input-height;
}

:deep(.el-slider__bar) {
  background-color: #8300bf;
}

:deep(.el-slider__button) {
  border-color: #8300bf;
}

.discrete {
  margin-left: 8px;
  width: calc(100% - 16px);
}

.discrete {
  :deep(.el-input__inner) {
    font-family: Asap, sans-serif;
    font-size: 16px;
  }
}

.discrete :deep(.el-input__inner:focus),
.discrete :deep(.el-input.is-focus .el-input__inner) {
  border-color: #8300bf;
}

.discrete-popper .el-select-dropdown__item {
  font-family: Asap, sans-serif;
}

.discrete-popper .el-select-dropdown__item.is-selected {
  font-weight: normal;
  color: #8300bf;
}

.scalar :deep(.el-input__inner) {
  text-align: center;
}

.scalar :deep(.el-input__wrapper) {
  padding-left: 4px;
  padding-right: 4px;
}

div.simulation-vuer {
  height: 100%;
}

div.sliders-and-fields {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  margin-top: -4px;
}

p.default {
  font-family: Asap, sans-serif;
  letter-spacing: 0;
  margin: 16px 0;
  text-align: start;
}

p.discrete {
  margin-top: 0;
  margin-bottom: 4px;
}

p.scalar {
  margin-bottom: 0;
  margin-top: 0;
  padding-right: $input-number-width;
}
</style>
