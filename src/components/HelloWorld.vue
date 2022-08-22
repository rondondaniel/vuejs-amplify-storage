<template>
  <v-container>
    <v-row class="justify-center pa-12">
      <v-col
          cols="12"
          sm="6"
          md="3"
      >
        <v-text-field
          v-model="projectName"
          label="Project Name"
          outlined
        ></v-text-field>   
      </v-col>
      <v-col
        cols="32"
        sm="16"
        md="8"
      >
        <v-file-input
          v-model="files"
          small-chips
          show-size
          multiple
          counter
          label="Drop Files Here"
          @change="onFileInput"
        ></v-file-input>
      </v-col>
    </v-row>
    <v-row class="justify-center pa-14">
      <v-btn
        color="primary"
        rounded
        @click="uploadFiles"
      >
        Upload Files
      </v-btn>
    
      <v-btn
        color="primary"
        rounded
        @click="testReactivity"
      >
        Test
      </v-btn>
    </v-row>
    <template>
      <v-data-table
        dense
        :items="progressInfo"
        :items-per-page="5"
        hide-default-header
        item-key="name"
        class="elevation-1"
      >
        <template v-slot:item="{ item }">
         <v-row>
          <v-col cols="6" align-self="start">
              {{ item.name }}
          </v-col>
          <v-col cols="1" align-self="center">
              {{ item.state }}
          </v-col>
          <v-col cols="1" align-self="center">
            <v-progress-linear
            :background-opacity="0.3"
            :buffer-value="100"
            :height="25"
            :width="150"
            :rounded="true"
            :value="item.percentage"
          >
            <strong>{{ item.percentage }}%</strong>
          </v-progress-linear>
          </v-col>
          <v-col cols="1" align-self="center">
              <v-btn 
                class="mx-2"
                fab
                dark
                small
                color="primary"
                @click="changeState(item.id, 'PAUSE')"
              >
                <v-icon dark>
                  mdi-pause
                </v-icon>
              </v-btn>
          </v-col>
          <v-col cols="1" align-self="center">
              <v-btn 
                class="mx-2"
                fab
                dark
                small
                color="error"
                @click="changeState(item.id, 'CANCEL')"
              >
                <v-icon dark>
                  mdi-close
                </v-icon>
              </v-btn>
          </v-col>
          </v-row>
        </template>      
      </v-data-table>
    </template>
  </v-container>
</template>

<script>
  import { Storage } from 'aws-amplify'

  export default {
    name: 'HelloWorld',

    data: () => ({
      files: [],
      projectName: null,
      progressInfo: [],
      fileState: {
        READY: 'READY',
        PENDING: 'PENDING',
        UPLOADING: 'UPLOADING',
        PAUSE: 'PAUSE',
        RESUME:'RESUME',
        CANCEL: 'CANCEL',
        DONE: 'DONE'
      },
      itemsPerPage: 4,
    }),
    methods: {
      onFileInput (_files) {
        this.progressInfo = [];

        for (let i = 0; i < _files.length; i++) {     
          let _value =  { id: i, name: _files[i].name, percentage: 0, state: this.fileState.READY }
          this.progressInfo.push(_value)
        }
        console.log('Init ProgressInfo:', this.progressInfo);
      },
      testReactivity () {
        let i = 1        
        this.changeState(i, this.fileState.PAUSE)        
        console.log(this.progressInfo)
      },
      uploadFiles: async function () {
        for (let i = 0; i < this.files.length; i++) {
          const upload = await Storage.put(this.projectName + "/" + this.files[i].name, this.files[i], {
            level: "private",
            contentType: this.files[i].type,
            resumable: true,
            completeCallback: (event) => {
              this.changeState(i, this.fileState.DONE, 100)
              console.log(`${this.files[i].name} Successfully uploaded ${event.key}`);
            },
            progressCallback: (progress) => {
              switch (this.progressInfo[i].state) {
                case this.fileState.PAUSE:
                  console.log("Paused file:", this.files[i].name);
                  upload.pause();
                  break;
                //case this.fileState.RESUME:
                //  console.log("Resuming file:", this.files[i].name);
                //  upload.resume();
                //  break;
                case this.fileState.CANCEL:
                  console.log("Canceled file:", this.files[i].name);
                  Storage.cancel(upload);
              }
              if (this.progressInfo[i].state == this.fileState.UPLOADING
                || this.progressInfo[i].state == this.fileState.READY
                || this.progressInfo[i].state == this.fileState.RESUME) {

                this.changeState(
                  i, 
                  this.fileState.UPLOADING, 
                  Math.ceil(progress.loaded / progress.total * 100));

                console.log(`${this.files[i].name} Uploaded: ${progress.loaded}/${progress.total}`);
              }              
            },
            errorCallback: (err) => {
              console.error(`${this.files[i].name}: Unexpected error while uploading`, err);
            }
          });
          
          console.log(upload.fileId);
          console.log(this.progressInfo);
        }
      },
      pause (index) {
        console.log("Pause file:", index);
        let bufferFile = this.files[index]
        this.progressInfo.splice(index, 1, {
          id: index,
          name: bufferFile.name,
          percentage: bufferFile.percentage,
          state: this.fileState.PAUSE
        })
      },
      cancel (index) {
        console.log("Cancel file:", index);
        let bufferFile = this.files[index]
        this.progressInfo.splice(index, 1, {
          id: index,
          name: bufferFile.name,
          percentage: bufferFile.percentage,
          state: this.fileState.CANCEL
        })
      },
      changeState (index, state, progress) {
        let bufferFile = this.progressInfo[index];
        
        if (progress === undefined) {
          progress = bufferFile.percentage;
        }

        if (state == this.fileState.PAUSE && bufferFile.state == this.fileState.PAUSE) {
          state = this.fileState.RESUME;
          this.uploadFiles();
        }
        
        this.progressInfo.splice(index, 1, {
          id: index,
          name: bufferFile.name,
          percentage: progress,
          state: state
        })

        console.log(state,' file:', bufferFile.name);
      },
    }
  }
</script>
