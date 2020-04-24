<template>
  <v-app>
    <v-snackbar v-model="snackbarShown">
      {{ snackbarText }}
    </v-snackbar>

    <v-dialog v-model="dialogShown" :max-width="dialogOptions.width" :style="{ zIndex: dialogOptions.zIndex }" @keydown.esc="cancel">
      <v-card>
        <v-toolbar dark :color="dialogOptions.color" dense flat>
          <v-toolbar-title class="white--text">{{ dialogTitle }}</v-toolbar-title>
        </v-toolbar>
        <v-card-text v-show="!!dialogMessage" class="pa-4">{{ dialogMessage }}</v-card-text>
        <v-card-actions class="pt-0">
          <v-spacer></v-spacer>
          <v-btn color="grey" text @click.native="dialogNo">No</v-btn>
          <v-btn color="primary darken-1" text @click.native="dialogYes">Yes</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-content>
      <v-container class="container">
        <header>
          <div class="langs">
            <v-select
              solo
              dense
              flat
              v-model="$root.$i18n.locale"
              v-bind:items="langs"
              prepend-inner-icon="mdi-web"
            />
          </div>
          <h1 v-html="$t('title')" />
        </header>

        <v-stepper v-model="step" vertical>
          <v-stepper-step v-bind:complete="step > 1" step="1">
            <span class="step">{{ $t('prepare:title') }}</span>
          </v-stepper-step>

          <v-stepper-content step="1">
            <div class="section" v-if="supported">
              <p v-html="$t('prepare:p1')" />
              <p>
                <ol>
                  <li v-html="$t('prepare:p1:li1')" />
                  <li v-html="$t('prepare:p1:li2')" />
                  <li v-html="$t('prepare:p1:li3')" />
                  <li v-html="$t('prepare:p1:li4')" />
                  <li v-html="$t('prepare:p1:li5')" />
                </ol>
              </p>
            </div>
            <div class="section" v-else>
              <p v-html="$t('prepare:unsupported:p1')" />
              <p v-html="$t('prepare:unsupported:p2')" />
            </div>
            <div class="nav" v-if="supported">
              <v-btn color="primary" depressed v-on:click="step = 2">{{ $t('prepare:next') }}</v-btn>
            </div>
          </v-stepper-content>

          <v-stepper-step v-bind:complete="step > 2" step="2" v-bind:rules="[() => supported]">
            <span class="step" v-if="!supported">{{ $t('connect:title:unsupported') }}</span>
            <span class="step" v-else-if="name">{{ $t('connect:title:connected', { name }) }}</span>
            <span class="step" v-else>{{ $t('connect:title') }}</span>
          </v-stepper-step>

          <v-stepper-content step="2">
            <div class="section">
              <p v-html="$t('connect:p1')" />
              <p v-html="$t('connect:p2')" />
            </div>
            <div class="nav">
              <v-btn color="primary" depressed v-on:click="connect">{{ $t('connect:connect') }}</v-btn>
              &nbsp;
              <v-btn color="primary" text v-on:click="step = 1">{{ $t('connect:previous') }}</v-btn>
            </div>
          </v-stepper-content>

          <v-stepper-step v-bind:complete="step > 3" step="3">
            <span class="step" v-if="selected">{{ $t('select:title:selected', { name: selected ? selected.name : null }) }}</span>
            <span class="step" v-else>{{ $t('select:title') }}</span>
          </v-stepper-step>

          <v-stepper-content step="3">
            <div class="section">
              <upload
                class="upload"
                ref="upload"
                accept="application/zip"
                extensions="zip"
                v-model="files"
                v-bind:drop="true"
                v-bind:drop-directory="false"
                v-on:input-file="selectFile"
              >
                <div class="upload-hint active" v-if="uploader && uploader.dropActive" v-html="$t('select:uploader:active')" />
                <div class="upload-hint selected" v-else-if="selected">{{selected ? selected.name : null}}</div>
                <div class="upload-hint" v-else v-html="$t('select:uploader:normal')" />
              </upload>
            </div>
            <div class="nav">
              <v-btn color="primary" depressed v-on:click="step = 4" v-bind:disabled="!selected">{{ $t('select:next') }}</v-btn>
              &nbsp;
              <v-btn color="primary" text v-on:click="step = 2">{{ $t('select:previous') }}</v-btn>
            </div>
          </v-stepper-content>

          <v-stepper-step v-bind:complete="step > 4" step="4">
            <span class="step">{{ $t('confirm:title') }}</span>
          </v-stepper-step>

          <v-stepper-content step="4">
            <div class="section" v-if="flashing">
              <p v-html="$t('confirm:flashing')" />
              <v-progress-linear
                v-bind:value="progress ? Math.round(progress / progressTotal * 100) : 0"
                v-bind:indeterminate="progress == 0"
              />
            </div>
            <div class="section" v-else>
              <p v-html="$t('confirm:p1')" />
            </div>
            <div class="nav" v-if="!flashing">
              <v-btn color="primary" depressed v-on:click="sideload">{{ $t('confirm:start') }}</v-btn>
              &nbsp;
              <v-btn color="primary" text v-on:click="step = 3">{{ $t('confirm:previous') }}</v-btn>
            </div>
          </v-stepper-content>

          <v-stepper-step v-bind:complete="step > 5" step="5">
            <span class="step">{{ $t('done:title') }}</span>
          </v-stepper-step>

          <v-stepper-content step="5">
            <div class="section">
              <p v-html="$t('done:p1')" />
            </div>
            <div class="nav">
              <v-btn color="primary" depressed v-on:click="reset">{{ $t('done:reset') }}</v-btn>
            </div>
          </v-stepper-content>
        </v-stepper>
      </v-container>
    </v-content>

    <footer>
      &copy;2020
    </footer>

  </v-app>
</template>

<script>
import Adb from 'webadb';
import upload from 'vue-upload-component';

import messages from './i18n';
import readFile from './utils/readFile';

const langs = Object.keys(messages)
  .sort()
  .map(lang => ({
    text: messages[lang].name,
    value: lang,
  }));

export default {
  name: 'App',
  components: {
    upload,
  },
  data() {
    return {
      langs: langs,
      supported: typeof navigator.usb != 'undefined',
      step: 1,
      name: null,
      files: [],
      uploader: null,
      selected: null,
      flashing: false,
      progress: 0,
      progressTotal: 0,
      snackbarShown: false,
      snackbarText: '',
      dialogShown: false,
      dialogResolve: null,
      dialogMessage: null,
      dialogTitle: null,
      dialogOptions: {
        color: 'primary',
        width: 290,
        zIndex: 200
      }
    };
  },
  watch: {
    step() {
      this.uploader = this.$refs.upload;
    },
  },
  methods: {
    showSnackbar(text) {
      this.snackbarText = text;
      this.snackbarShown = true;
    },
    showDialog(title, message, options) {
      this.dialogShown = true
      this.dialogTitle = title
      this.dialogMessage = message
      this.dialogOptions = Object.assign(this.dialogOptions, options)
      return new Promise((resolve) => {
        this.dialogResolve = resolve
      })
    },
    dialogYes() {
      this.dialogResolve(true)
      this.dialogShown = false
    },
    dialogNo() {
      this.dialogResolve(false)
      this.dialogShown = false
    },
    selectFile(file) {
      if (!file) {
        this.selected = null;
      } else if (!file.name.endsWith('.zip')) {
        this.selected = null;
        this.showSnackbar(this.$t('hint:zip'));
      } else {
        this.selected = file.file;
        this.step = 3;
      }
    },
    async connect() {
      try {
        this.usb = await Adb.open('WebUSB');
        this.adb = await this.usb.connectAdb('host::');
        if (this.adb.mode != 'sideload') {
          if (await this.showDialog(this.$t('prompt:reboot:title'),
                                    this.$t('prompt:reboot:message'))) {
            await this.adb.reboot('sideload');
          }
          this.usb.close();
          return;
        }

        const { manufacturerName, productName } = this.usb.device;
        this.name = `${manufacturerName} ${productName}`;
        this.step = 3;
      } catch (e) {
        console.error(e);
        if (e.name != 'NotFoundError') {
          this.showSnackbar(this.$t('hint:connect'));
        }
      }
    },
    async sideload() {
      this.flashing = true;
      this.progress = 0;

      const chunk_size = 64 * 1024;

      const content = await readFile(this.selected);
      const stream = await this.adb.open(`sideload-host:${content.length}:${chunk_size}`);

      this.progressTotal = content.length;

      while (this.flashing) {
        const response = await stream.receive();

        if (response.cmd == 'OKAY') {
          await stream.send('OKAY');
        }

        if (response.cmd != 'WRTE') {
          continue;
        }

        const result = new TextDecoder("utf-8").decode(response.data);
        if (result == 'DONEDONE' || result == 'FAILFAIL')
          break;

        const start = parseInt(result) * chunk_size;
        let end = start + chunk_size;
        if (end > content.length) {
          end = content.length;
        }

        const data = content.slice(start, end);

        await stream.send('WRTE', data);
        await stream.send('OKAY');

        this.progress += data.length;

        if (this.progress >= this.progressTotal)
          break;
      }
      this.step = 5;
      this.flashing = false;
    },
    reset() {
      this.name = null;
      this.selected = null;
      this.progress = 0;
      this.progressTotal = 0;
      this.step = 1;
    },
  },
};
</script>

<style>
h1 {
  min-height: 38px;
  line-height: 38px;
  font-size: 24px;
  font-weight: normal;
  margin: 0 0 10px 10px;
}

h1 small {
  font-size: 14px;
}

.langs {
  max-width: 180px;
  height: 38px;
  float: right;
  font-size: 12px;
  margin-right: 4px;
}

footer {
  font-size: 12px;
  color: #999;
  margin: auto auto 20px;
}

.container {
  margin: auto;
  max-width: 600px;
}

.step {
  margin-left: 4px;
  font-size: 15px;
  font-weight: bold;
}

.section {
  margin: 4px 4px 20px;
}

.nav {
  margin: 4px;
}

.upload {
  display: block;
  width: 100%;
}

.upload * {
  cursor: pointer;
}

.upload-hint {
  text-align: center;
  padding: 40px 20px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  border: 2px dotted #eee;
  border-radius: 8px;
  transition: background, border-color 300ms;
}

.upload-hint.active {
  background: #BBDEFB;
  border-color: #64B5F6;
}

.upload-hint.selected {
  border-color: #1E88E5;
}

.v-select__selection--comma {
  margin: 7px 4px 7px 7px !important;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  width: 100%;
  text-align: center;
}
</style>
