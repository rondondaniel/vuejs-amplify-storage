<template>
  <v-container>
    <v-row class="justify-center pa-4">
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
    <v-row class="justify-center pa-4">
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
        @click="listFiles"
      >
        Refresh File List
      </v-btn>
    </v-row>
    <template>
      <v-data-table
        dense
        :items="progressInfo"
        :items-per-page="5"
        hide-default-header
        item-key="name"
        class="elevation-1 pa-14"
      >
        <template v-slot:item="{ item }">
         <v-row>
          <v-col cols="5" align-self="start">
            {{ item.name }}
          </v-col>
          <v-col cols="1" align-self="center">
            {{ item.state }}
          </v-col>
          <v-col cols="2" align-self="center">
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
            <v-col cols="2" align-self="center">
              <v-btn 
                class="mx-2"
                fab
                dark
                small
                color="primary"
                :disabled="item.state == fileState.CANCEL"
                @click="(item.state == fileState.PAUSE) ? resume(item.id) : pause(item.id)"
              >
                <v-icon dark>
                  {{ (item.state == fileState.PAUSE) ? 'mdi-play' : 'mdi-pause' }}
                </v-icon>
              </v-btn>
              <v-btn 
                class="mx-2"
                fab
                dark
                small
                :color="item.state == fileState.DONE ? 'success' : 'error'"
                @click="item.state == fileState.DONE ? {} : cancel(item.id)"
              >
                <v-icon dark>
                  {{ (item.state == fileState.DONE ? 'mdi-cloud-check' : 'mdi-close' ) }}
                </v-icon>
              </v-btn>
            </v-col>
          </v-row>
        </template>      
      </v-data-table>
    </template>
    <v-divider></v-divider>
    <template>
      <v-data-table
        dense
        :items="listOfFiles"
        :items-per-page="5"
        hide-default-header
        item-key="key"
        class="elevation-1 pa-14"
      >
        <template v-slot:item="{ item }">
          <v-row>
            <v-col cols="5" align-self="start">
              {{ item.name }}
            </v-col>
            <v-col cols="1" align-self="center">
              {{ item.size }}
            </v-col>
            <v-col cols="2" align-self="center">
              {{ item.modified }}
            </v-col>
            <v-col cols="2" align-self="center">
              <v-btn
                class="mx-2"
                fab
                dark
                small
                color="success"
                :href="item.url"
                target="_blank"
              >
                <v-icon>
                  mdi-cloud-download
                </v-icon>
              </v-btn>
              <v-btn
                class="mx-2"
                fab
                dark
                small
                color="error"
                @click="removeFile(item.name)"
              >
                <v-icon>
                  mdi-delete
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
    name: 'UploadFiles',

    data: () => ({
      files: [],
      projectName: null,
      progressInfo: [],
      itemsPerPage: 4,
      upload: [],
      fileState: {
        READY: 'READY',
        PENDING: 'PENDING',
        UPLOADING: 'UPLOADING',
        PAUSE: 'PAUSED',
        RESUME:'RESUME',
        CANCEL: 'CANCELED',
        DONE: 'DONE'
      },
      listOfFiles: [],
    }),
    methods: {
      onFileInput (_files) {
        this.progressInfo = [];
 
        for (let i = 0; i < _files.length; i++) {     
          let _value =  { id: i, name: _files[i].name, percentage: 0, state:  this.fileState.READY}
          this.progressInfo.push(_value)
        }
      },
      async uploadFiles () {
        for (let i = 0; i < this.files.length; i++) {
          this.upload[i] = await Storage.put(this.projectName + "/" + this.files[i].name, this.files[i], {
            level: "private",
            contentType: this.files[i].type,
            resumable: true,
            completeCallback: (event) => {
              this.changeState(i, this.fileState.DONE, 100);
              console.log(`${this.files[i].name} Successfully uploaded ${event.key}`);
              this.listFiles()
            },
            progressCallback: (progress) => {
              this.changeState(
                i,
                this.fileState.UPLOADING,
                Math.ceil(progress.loaded / progress.total * 100)
              );              
              console.log(`${this.files[i].name} Uploaded: ${progress.loaded}/${progress.total}`);
            },
            errorCallback: (err) => {
              console.error(`${this.files[i].name}: Unexpected error while uploading`, err);
            }
          });
        }
      },
      pause (index) {
        this.upload[index].pause();
        this.changeState(index, this.fileState.PAUSE);
      },
      resume (index) {
        this.upload[index].resume();
        this.changeState(index, this.fileState.RESUME);
      },
      async cancel (index) {
        try {
          await Storage.cancel(this.upload[index]);
          this.changeState(index, this.fileState.CANCEL, 0);
        } 
        catch (err) {
          console.log('Oops something wrong with canceling upload: ', err);
        }
      },
      changeState (index, newState, progress) {
        let bufferFile = this.progressInfo[index];
        
        if (progress === undefined) {
          progress = bufferFile.percentage;
        }
        
        this.progressInfo.splice(index, 1, {
          id: index,
          name: bufferFile.name,
          percentage: progress,
          state: newState
        })
      },
      async listFiles () {
        this.listOfFiles = [];

        await Storage.list(this.projectName + '/', { level: 'private' })
          .then(result => {
            for (let i in result) {
              let value =  { 
                id: i, 
                name: result[i].key, 
                modified: result[i].lastModified, 
                size: result[i].size,
                url: '',
              }
              this.listOfFiles.push(value)
            }
          })
          .catch(err => console.log(err));
        this.signedURL();

      },
      async signedURL () {
        for (let i = 0; i < this.listOfFiles.length; i++) {   
          let bufferFile = this.listOfFiles[i];
  
          await Storage.get(bufferFile.name, { level: 'private' })
            .then(result => {
              this.listOfFiles.splice(i, 1, {
                id: bufferFile.id, 
                name: bufferFile.name, 
                modified: bufferFile.modified, 
                size: bufferFile.size,
                url: result,
              })
            })
            .catch(err => console.log(err));
        }
      },
      async removeFile(fileName) {
        try {
          await Storage.remove(fileName, { level: 'private' })
            .catch(err => console.log(err));
        }
        catch (err) {
          console.log('Oops something wrong with removing file: ', err);
        }
        this.listFiles();
      },
    }
  }
</script>
