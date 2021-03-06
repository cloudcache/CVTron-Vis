<template>
  <v-stepper v-model="current_step" vertical>
    <v-stepper-step step="1" :complete="current_step > 1">
      Choose a Task
      <small>You want classification, detection or segmentation?</small>
    </v-stepper-step>
    <v-stepper-content step="1">
      <v-layout row>
        <v-btn color="info" @click="choose_task_type('classifier')">Classification</v-btn>
        <v-btn color="info" @click="choose_task_type('detector')">Detection</v-btn>
        <v-btn color="info" @click="choose_task_type('segmentor')">Segmentation</v-btn>
      </v-layout>
    </v-stepper-content>
    <v-stepper-step step="2" :complete="current_step > 2">Config a Network Structure
        <small>Use existed models or create your own</small></v-stepper-step>
    <v-stepper-content step="2">
      <v-layout row>
      <v-btn color="info" @click="choose_model_type('predefined')">Pre-Defined</v-btn>
      <v-btn color="info" @click="choose_model_type('upload')">Upload</v-btn>
      <br/>
      <v-select
          v-if="isPredefined"
          :items="items"
          label="Selected Model"
        ></v-select>
      <a href="https://cvtron.unarxiv.org/zh/guide/data-format.html" class="upload_guide">Upload Guidelines</a>
      </v-layout>
      <v-layout row v-for="(key, value) in config" :key="value">
      <v-flex xs4>
        <v-subheader>{{ value }}</v-subheader>
      </v-flex>
      <v-flex xs8>
        <v-text-field
          name="input-1"
          :label=value
          :id=value
          v-model=config[value]
        ></v-text-field>
      </v-flex>
      </v-layout>
      <v-btn v-if="step_3_continue_visibility" color="info" @click="step_forward()">Continue</v-btn>
      <v-btn v-if="step_3_continue_visibility" flat @click="step_back()">Cancel</v-btn>
    </v-stepper-content>
    <v-stepper-step step="3" :complete="current_step > 3">Upload Dataset
    </v-stepper-step>
    <v-stepper-content step="3">
          <v-btn v-model="filename"
                  @click.native="onFocus"
                  ref="fileTextField">UPLOAD</v-btn>
                  <br/>
      <input type="file" ref="fileInput" v-on:change="onFileChange">
      <v-btn flat @click="step_back()">Cancel</v-btn>
    </v-stepper-content>
    <v-stepper-step step="4">Monitor Training Logs
        <small>View your training process</small>
    </v-stepper-step>
    <v-stepper-content step="4">
      <Train :url="log_url" v-if="log_flag"></Train>
      <v-btn color="primary" @click.native="current_step = 1">Complete</v-btn>
    </v-stepper-content>
  </v-stepper>
</template>

<script>
import Train from '@/components/Tasks/Train'
import { getTrainConfig, startTrain, upload, getModels } from '@/services'

export default {
  data () {
    return {
      filename: '',
      current_step: 1,
      formdata: '',
      result: [],
      log_url: '',
      items: [],
      config: {},
      log_flag: true,
      step_3_continue_visibility: false,
      isPredefined: false
    }
  },
  components: {
    Train
  },
  mounted () {
  },
  methods: {
    choose_task_type (taskName) {
      this.task_type = taskName
      this.current_step = 2
    },
    choose_model_type (modelType) {
      let self = this
      this.model_type = modelType
      if (this.model_type === 'predefined') {
        getModels().then(function (res) {
          self.items = res.data.filter(function (each) {
            if (each.task === self.task_type) {
              return each
            }
          }).map(function (each) {
            return each.name
          })
          console.log(self.items)
        })
        this.isPredefined = true
        getTrainConfig(this.task_type).then(function (res) {
          self.config = res.data
          self.step_3_continue_visibility = true
        })
      } else {
        this.isPredefined = false
      }
    },
    onFileChange (e) {
      let file = e.target.files[0]
      let endpoint = this.task_type + '/upload'
      let self = this
      upload(endpoint, file).then(function (res) {
        self.train_task_id = res.data.file_id
        let trainFinalConfig = {
          'config': self.config,
          'file': self.train_task_id
        }
        self.log_url = res.data.file_id + '/log.json'
        self.log_flag = true
        startTrain(self.task_type, trainFinalConfig)
        self.current_step = 4
      })
    },
    onFocus () {
      if (!this.disabled) {
        this.$refs.fileInput.click()
      }
    },
    step_back () {
      if (this.current_step >= 1) {
        this.current_step -= 1
      }
    },
    step_forward () {
      if (this.current_step <= 4) {
        this.current_step += 1
      }
    }
  }
}
</script>

<style>
.upload_guide {
  margin-top: auto;
  margin-bottom: auto;
}
</style>
